# filament 4
```sh
composer require filament/filament:"~4.0"
```
```sh
php artisan filament:install --panels
```
```sh
php artisan migrate
```
```sh
php artisan make:filament-user
```


# filament shield
```sh
composer require bezhansalleh/filament-shield
```
```sh
php artisan vendor:publish --tag="filament-shield-config"
```

### app\Model\User
```sh
use Spatie\Permission\Traits\HasRoles;

...

class User extends Authenticatable
{
    use HasRoles;
}
```

```sh
php artisan make:filament-resource User
```


```sh
php artisan shield:setup
```
```sh
php artisan shield:install admin
```

```sh
php artisan shield:super-admin
```

```sh
php artisan shield:generate --all --panel=admin
```

### criar a model evendot
```sh
php artisan make:model Evento -m
```
### Migration
#### Arquivo: database/migrations/xxxx_create_eventos_table.php
```sh
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('eventos', function (Blueprint $table) {
            $table->id();
            $table->string('titulo');
            $table->dateTime('starts_at');
            $table->dateTime('ends_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('eventos');
    }
};
```

```sh
php artisan migrate
```
### Model
#### Arquivo: app/Models/Evento.php
```sh
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Evento extends Model
{
    protected $fillable = ['titulo', 'starts_at', 'ends_at'];

    protected $casts = [
        'starts_at' => 'datetime',
        'ends_at' => 'datetime',
    ];
}
```


### quando criar usuario selecionar o tipo
#### app/Filament/Resources/User/Schemas/UserForm.php

```sh
use Filament\Forms\Components\Select;

...

Select::make('roles')
    ->relationship('roles', 'name')
    ->multiple()
    ->preload()
    ->searchable(),
```

### mostrar o tipo de usuario na listagem
#### app/Filament/Resources/User/Tables/UsersTable.php
```sh
...

TextColumn::make('roles_badge')
                ->label('Tipo')
                ->state(fn ($record) => $record->getRoleNames()->implode(',')) // "admin,editor"
                ->badge()
                ->separator(','), // vira múltiplos badges
```

# full calendar para filament 4
```sh
composer require saade/filament-fullcalendar:"v4.0.0-beta2" -W
```

### criar widget do full calendar
#### Selecione o "Custom" apos rodar o comando abaixo
```sh
php artisan make:filament-widget CalendarWidget --panel=admin
```


#### Registre o plugin
#### app/Providers/Filament/AdminPanelProvider.php
```sh
use Saade\FilamentFullCalendar\FilamentFullCalendarPlugin;

...

->plugins([
    FilamentShieldPlugin::make(),
    FilamentFullCalendarPlugin::make()
        ->selectable() // <- ESSENCIAL para selecionar dias/horários
        ->editable()   // opcional: arrastar/redimensionar eventos
])
```

### app/Filament/Widget/CalendarWidget
```sh
<?php

namespace App\Filament\Widgets;

use App\Models\Agenda;
use BezhanSalleh\FilamentShield\Traits\HasWidgetShield;
use Carbon\Carbon;
use Filament\Forms\Components\Hidden;
use Filament\Forms\Components\Placeholder;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Utilities\Set;
use Filament\Schemas\Schema;
use Filament\Support\Exceptions\Halt;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\ValidationException;
use Saade\FilamentFullCalendar\Actions;
use Saade\FilamentFullCalendar\Widgets\FullCalendarWidget;

class CalendarWidget extends FullCalendarWidget
{
    use HasWidgetShield;

    public Model|string|null $model = Agenda::class;

    public static function getHeading(): string
    {
        return 'Calendário';
    }

    public function config(): array
    {
        return [
            'selectable' => true,
            'unselectAuto' => true,
            'firstDay' => 1,

            // ✅ 24h
            'locale' => 'pt-br',
            'slotLabelFormat' => [
                'hour' => '2-digit',
                'minute' => '2-digit',
                'hour12' => false,
            ],
            'eventTimeFormat' => [
                'hour' => '2-digit',
                'minute' => '2-digit',
                'hour12' => false,
            ],
        ];
    }

    /**
     * ✅ Chamado pelo JS do dateClick antes de abrir o modal.
     * Retorna true se o dia tiver ao menos 1 horário livre; senão notifica e retorna false.
     */
    public function checkDayHasHours(string $dateStr): bool
    {
        $dia = Carbon::parse($dateStr)->toDateString();

        $options = $this->availableHourOptions($dia);

        if (empty($options)) {
            $this->notifySemHorario($dia);
            return false;
        }

        return true;
    }

    public function dateClick(): string
    {
        return <<<JS
            function(info) {
                info.view.calendar.select(info.dateStr);

                // ✅ Antes de abrir modal, valida com o backend se há horários no dia
                info.view.calendar.el.__livewire.call('checkDayHasHours', info.dateStr)
                    .then(function(ok) {
                        if (!ok) {
                            info.view.calendar.unselect();
                            return;
                        }

                        info.view.calendar.el.__livewire.mountAction('create', {
                            start: info.dateStr,
                            end: info.dateStr
                        });
                    });
            }
        JS;
    }

    private function notifySemHorario(string $dia): void
    {
        $data = Carbon::parse($dia)->format('d/m/Y');

        Notification::make()
            ->title('Sem horários disponíveis')
            ->body("No dia {$data} não há horário disponível.")
            ->warning()
            ->send();
    }

    private function notifyHorarioAjustado(string $dia, string $horaSelecionada): void
    {
        $data = Carbon::parse($dia)->format('d/m/Y');

        Notification::make()
            ->title('Horário ajustado')
            ->body("O horário do arrasto não está disponível em {$data}. Preenchi {$horaSelecionada} (primeiro horário livre). Você pode alterar no campo Horário.")
            ->warning()
            ->send();
    }

    /**
     * Base: 08-11 e 14-17 (12h removido)
     */
    private function baseHourOptions(): array
    {
        $hours = array_merge(range(8, 11), range(14, 17));

        $options = [];
        foreach ($hours as $h) {
            $options[sprintf('%02d:00', $h)] = sprintf('%02d:00', $h);
        }

        return $options;
    }

    /**
     * Remove horas já ocupadas no dia.
     */
    private function availableHourOptions(?string $dia, ?int $ignoreEventoId = null): array
    {
        $options = $this->baseHourOptions();

        if (! $dia) {
            return $options;
        }

        $ocupados = Agenda::query()
            ->whereDate('starts_at', $dia)
            ->when($ignoreEventoId, fn ($q) => $q->whereKeyNot($ignoreEventoId))
            ->pluck('starts_at')
            ->map(fn ($dt) => Carbon::parse($dt)->format('H:00'))
            ->unique()
            ->all();

        foreach ($ocupados as $hora) {
            unset($options[$hora]);
        }

        return $options;
    }

    public function fetchEvents(array $fetchInfo): array
    {
        $start = Carbon::parse($fetchInfo['start']);
        $end = Carbon::parse($fetchInfo['end']);

        return Agenda::query()
            ->where('starts_at', '<', $end)
            ->where(function ($q) use ($start) {
                $q->whereNull('ends_at')->orWhere('ends_at', '>', $start);
            })
            ->get()
            ->map(fn (Agenda $e) => [
                'id' => (string) $e->id,
                'title' => $e->titulo,
                'start' => $e->starts_at,
                'end' => $e->ends_at,
            ])
            ->all();
    }

    public function getFormSchema(): array
    {
        return [
            Hidden::make('evento_id')->dehydrated(false),
            Hidden::make('dia')->dehydrated(false),

            TextInput::make('titulo')
                ->label('Título')
                ->required()
                ->maxLength(255),

            Placeholder::make('data_visual')
                ->label('Data')
                ->content(fn (Get $get) => $get('dia') ?: '-'),

            Select::make('hora_inicio')
                ->label('Horário')
                ->options(fn (Get $get) => $this->availableHourOptions(
                    $get('dia'),
                    $get('evento_id')
                ))
                ->required()
                ->live()
                ->afterStateUpdated(function (?string $state, Set $set, Get $get) {
                    $dia = $get('dia');

                    if (! $dia || ! $state) {
                        return;
                    }

                    $inicio = Carbon::parse("{$dia} {$state}");
                    $fim = $inicio->copy()->addHour();

                    $set('starts_at', $inicio->toDateTimeString());
                    $set('ends_at', $fim->toDateTimeString());
                }),

            Hidden::make('starts_at')->required(),
            Hidden::make('ends_at')->required(),
        ];
    }

    protected function headerActions(): array
    {
        return [
            Actions\CreateAction::make()
                ->label('Agendar')
                ->mountUsing(function (Schema $form, array $arguments) {
                    $dia = isset($arguments['start'])
                        ? Carbon::parse($arguments['start'])->toDateString()
                        : null;

                    if (! $dia) {
                        Notification::make()
                            ->title('Selecione um dia')
                            ->body('Clique em um dia no calendário para agendar.')
                            ->warning()
                            ->send();

                        throw new Halt();
                    }

                    $options = $this->availableHourOptions($dia);

                    if (empty($options)) {
                        $this->notifySemHorario($dia);
                        throw new Halt();
                    }

                    $hora = array_key_first($options);

                    $inicio = Carbon::parse("{$dia} {$hora}");
                    $fim = $inicio->copy()->addHour();

                    $form->fill([
                        'evento_id' => null,
                        'dia' => $dia,
                        'hora_inicio' => $hora,
                        'starts_at' => $inicio->toDateTimeString(),
                        'ends_at' => $fim->toDateTimeString(),
                    ]);
                })
                ->mutateFormDataUsing(function (array $data): array {
                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Agenda::query()
                        ->whereDate('starts_at', $start->toDateString())
                        ->whereTime('starts_at', $start->format('H:i:s'))
                        ->exists();

                    if ($jaExiste) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Este horário já foi agendado. Selecione outro.',
                        ]);
                    }

                    unset($data['dia'], $data['hora_inicio'], $data['evento_id']);

                    return $data;
                }),
        ];
    }

    protected function modalActions(): array
    {
        return [
            Actions\EditAction::make()
                ->mountUsing(function (Schema $form, Model $record, array $arguments) {
                    /** @var Agenda $record */

                    $startArg = data_get($arguments, 'event.start');
                    $endArg   = data_get($arguments, 'event.end');

                    $isDragDrop = (bool) $startArg;

                    $start = $startArg
                        ? Carbon::parse($startArg)
                        : Carbon::parse($record->starts_at);

                    $end = $endArg
                        ? Carbon::parse($endArg)
                        : ($record->ends_at
                            ? Carbon::parse($record->ends_at)
                            : $start->copy()->addHour());

                    $dia = $start->toDateString();

                    // ✅ Se veio de drag&drop:
                    // 1) se o DIA não tem nenhum horário livre: avisa e cancela (evento volta após refresh)
                    // 2) se o HORÁRIO do drop não está disponível: preenche o primeiro horário livre (usuário pode mudar no select)
                    if ($isDragDrop) {
                        $options = $this->availableHourOptions($dia, (int) $record->getKey());

                        if (empty($options)) {
                            $this->notifySemHorario($dia);

                            // força re-render pra voltar visualmente o evento
                            $this->dispatch('$refresh');

                            throw new Halt();
                        }

                        $horaDesejada = $start->format('H:00');

                        if (! array_key_exists($horaDesejada, $options)) {
                            $horaSelecionada = array_key_first($options);

                            $start = Carbon::parse("{$dia} {$horaSelecionada}");
                            $end = $start->copy()->addHour();

                            $this->notifyHorarioAjustado($dia, $horaSelecionada);
                        }
                    }

                    $hora = $start->format('H:00');

                    $form->fill([
                        'evento_id' => $record->getKey(),
                        'dia' => $dia,
                        'hora_inicio' => $hora,
                        'titulo' => $record->titulo,
                        'starts_at' => $start->toDateTimeString(),
                        'ends_at' => $end?->toDateTimeString(),
                    ]);
                })
                ->mutateFormDataUsing(function (array $data, Model $record): array {
                    /** @var Agenda $record */
                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Agenda::query()
                        ->whereKeyNot($record->getKey())
                        ->whereDate('starts_at', $start->toDateString())
                        ->whereTime('starts_at', $start->format('H:i:s'))
                        ->exists();

                    if ($jaExiste) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Este horário já foi agendado. Selecione outro.',
                        ]);
                    }

                    unset($data['dia'], $data['hora_inicio'], $data['evento_id']);

                    return $data;
                }),

            Actions\DeleteAction::make(),
        ];
    }
}


```

```sh
php artisan shield:generate --all --panel=admin
```


