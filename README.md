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

use App\Models\Evento;
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
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\ValidationException;
use Saade\FilamentFullCalendar\Actions;
use Saade\FilamentFullCalendar\Widgets\FullCalendarWidget;

class CalendarWidget extends FullCalendarWidget
{
    use HasWidgetShield;

    public Model|string|null $model = Evento::class;

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

            // ✅ habilita drag & drop
            'editable' => true,

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
     * ✅ Hook do FullCalendar para arrasto com SNAP antes de abrir modal.
     */
    public function eventDrop(): string
    {
        return <<<JS
            function(info) {
                const lw = info.view.calendar.el.__livewire;
                const plain = info.event.toPlainObject();

                lw.call('validateDrop', {
                    id: plain.id,
                    start: plain.start,
                    end: plain.end
                }).then(function(result) {

                    // ❌ Sem horários no dia: reverte e (com delay) refaz fetch
                    if (!result || result.status === 'reject') {
                        info.revert();

                        setTimeout(function () {
                            info.view.calendar.refetchEvents();
                        }, 350);

                        return;
                    }

                    // ✅ SNAP visual IMEDIATO (para ok e adjust)
                    if (result.start) {
                        info.event.setStart(result.start);
                        plain.start = result.start;
                    }
                    if (result.end) {
                        info.event.setEnd(result.end);
                        plain.end = result.end;
                    }

                    // ✅ abre modal de edição já com dados ajustados (usuário pode trocar o horário)
                    lw.mountAction('edit', { event: plain });

                }).catch(function() {
                    info.revert();

                    setTimeout(function () {
                        info.view.calendar.refetchEvents();
                    }, 350);
                });
            }
        JS;
    }

    /**
     * ✅ Valida um drop (arrasto) e devolve o horário "snapado".
     * - reject: não tem horários no dia
     * - adjust: horário do drop não está disponível → ajusta para 1º horário livre
     * - ok: mantém o horário (normalizado para HH:00)
     */
    public function validateDrop(array $payload): array
    {
        $id = (int) ($payload['id'] ?? 0);
        $startRaw = $payload['start'] ?? null;

        if (! $startRaw) {
            return ['status' => 'reject'];
        }

        $start = Carbon::parse($startRaw);
        $dia = $start->toDateString();

        $options = $this->availableHourOptions($dia, $id);

        if (empty($options)) {
            $this->notifySemHorario($dia);
            return ['status' => 'reject'];
        }

        // Trabalhamos com blocos HH:00
        $horaDesejada = $start->format('H:00');

        // Se caiu num horário fora do permitido ou já ocupado, ajusta pro primeiro livre
        if (! array_key_exists($horaDesejada, $options)) {
            $horaSelecionada = array_key_first($options);

            $inicio = Carbon::parse("{$dia} {$horaSelecionada}");
            $fim = $inicio->copy()->addHour();

            $this->notifyHorarioAjustado($dia, $horaSelecionada);

            return [
                'status' => 'adjust',
                'start' => $inicio->toIso8601String(),
                'end' => $fim->toIso8601String(),
            ];
        }

        // ok: normaliza para HH:00 e 1h de duração
        $inicio = Carbon::parse("{$dia} {$horaDesejada}");
        $fim = $inicio->copy()->addHour();

        return [
            'status' => 'ok',
            'start' => $inicio->toIso8601String(),
            'end' => $fim->toIso8601String(),
        ];
    }

    public function dateClick(): string
    {
        return <<<JS
            function(info) {
                // ✅ NÃO chamar calendar.select() aqui

                // ✅ sempre abre o modal; se não houver horário, o modal exibirá o aviso e bloqueará salvar
                info.view.calendar.el.__livewire.mountAction('create', {
                    start: info.dateStr,
                    end: info.dateStr
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

        $ocupados = Evento::query()
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

        return Evento::query()
            ->where('starts_at', '<', $end)
            ->where(function ($q) use ($start) {
                $q->whereNull('ends_at')->orWhere('ends_at', '>', $start);
            })
            ->get()
            ->map(fn (Evento $e) => [
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
                ->maxLength(255)
                // ✅ desabilita tudo se não houver horário (starts_at vazio)
                ->disabled(fn (Get $get) => blank($get('starts_at'))),

            Placeholder::make('data_visual')
                ->label('Data')
                ->content(fn (Get $get) => $get('dia') ?: '-'),

            // ✅ aviso dentro do modal quando não existir horário livre
            Placeholder::make('sem_horario_aviso')
                ->label('')
                ->content(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return null;
                    }

                    $options = $this->availableHourOptions($dia, $get('evento_id'));
                    if (! empty($options)) {
                        return null;
                    }

                    $data = Carbon::parse($dia)->format('d/m/Y');
                    return "⚠️ No dia {$data} não há horários disponíveis.";
                })
                ->visible(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return false;
                    }

                    $options = $this->availableHourOptions($dia, $get('evento_id'));
                    return empty($options);
                }),

            Select::make('hora_inicio')
                ->label('Horário')
                ->options(fn (Get $get) => $this->availableHourOptions(
                    $get('dia'),
                    $get('evento_id')
                ))
                // ✅ se não há opções, desabilita e NÃO exige
                ->disabled(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return false;
                    }

                    return empty($this->availableHourOptions($dia, $get('evento_id')));
                })
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return true;
                    }

                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                })
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

            Hidden::make('starts_at')
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return true;
                    }

                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                }),

            Hidden::make('ends_at')
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) {
                        return true;
                    }

                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                }),

            /**
             * ✅ Desabilita o botão "Salvar" do modal quando starts_at estiver vazio
             * (sem depender de modalSubmitAction, que na sua versão está vindo null).
             */
            Placeholder::make('disable_submit_when_no_hours')
                ->label('')
                ->content('')
                ->extraAttributes([
                    'style' => 'display:none',
                    'x-data' => '{}',
                    'x-effect' => <<<'JS'
                        const root =
                            $el.closest('.fi-modal') ||
                            $el.closest('[role="dialog"]') ||
                            document;

                        const submit = root.querySelector('button[type="submit"]');
                        const startsAt = $wire.get('mountedActionsData.0.starts_at');

                        if (submit) {
                            submit.disabled = !startsAt;
                        }
                    JS,
                ]),
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

                    // Se abriu sem dia (botão), mantém aviso
                    if (! $dia) {
                        Notification::make()
                            ->title('Selecione um dia')
                            ->body('Clique em um dia no calendário para agendar.')
                            ->warning()
                            ->send();

                        // ainda abre o modal, mas sem dia não faz sentido: deixa vazio
                        $form->fill([
                            'evento_id' => null,
                            'dia' => null,
                            'hora_inicio' => null,
                            'starts_at' => null,
                            'ends_at' => null,
                        ]);

                        return;
                    }

                    $options = $this->availableHourOptions($dia);

                    // ✅ se não há horário: preenche tudo null (isso trava tudo no form)
                    if (empty($options)) {
                        $this->notifySemHorario($dia);

                        $form->fill([
                            'evento_id' => null,
                            'dia' => $dia,
                            'hora_inicio' => null,
                            'starts_at' => null,
                            'ends_at' => null,
                        ]);

                        return;
                    }

                    // ✅ pega o primeiro horário disponível do dia
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
                    // ✅ se não tem starts_at, é porque não tinha horário disponível
                    if (empty($data['starts_at'])) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Não há horário disponível para este dia.',
                        ]);
                    }

                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Evento::query()
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
                    /** @var Evento $record */

                    $startArg = data_get($arguments, 'event.start');
                    $endArg   = data_get($arguments, 'event.end');

                    $start = $startArg
                        ? Carbon::parse($startArg)
                        : Carbon::parse($record->starts_at);

                    $end = $endArg
                        ? Carbon::parse($endArg)
                        : ($record->ends_at
                            ? Carbon::parse($record->ends_at)
                            : $start->copy()->addHour());

                    $dia = $start->toDateString();
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
                    /** @var Evento $record */
                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Evento::query()
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

1) Migration: adicionar user_id em eventos

Crie a migration:

```sh
php artisan make:migration add_user_id_to_eventos_table --table=eventos
```

database/migrations/xxxx_xx_xx_xxxxxx_add_user_id_to_eventos_table.php
```sh
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::table('eventos', function (Blueprint $table) {
            $table->foreignId('user_id')
                ->after('id')
                ->constrained()
                ->cascadeOnDelete();
        });
    }

    public function down(): void
    {
        Schema::table('eventos', function (Blueprint $table) {
            $table->dropConstrainedForeignId('user_id');
        });
    }
};
```

2) Models: relacionamento + fillable
2.1) app/Models/Evento.php

```sh
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Evento extends Model
{
    protected $fillable = ['user_id', 'titulo', 'starts_at', 'ends_at'];

    protected $casts = [
        'starts_at' => 'datetime',
        'ends_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

2.2) app/Models/User.php (apenas adicione o método)

Adicione no final da classe:

```sh
use Illuminate\Database\Eloquent\Relations\HasMany;

public function eventos(): HasMany
{
    return $this->hasMany(Evento::class);
}
```

3) Widget do select de usuário no Dashboard (Filament v4)
3.1) Criar o widget

```sh
php artisan make:filament-widget SelecionarUsuarioAgendaWidget --panel=admin
```

3.2) Substituir o arquivo gerado por este

app/Filament/Widgets/SelecionarUsuarioAgendaWidget.php

```sh
<?php

namespace App\Filament\Widgets;

use App\Models\User;
use Filament\Forms;
use Filament\Schemas\Schema;
use Filament\Widgets\Widget;

class SelecionarUsuarioAgendaWidget extends Widget implements Forms\Contracts\HasForms
{
    use Forms\Concerns\InteractsWithForms;

    // ✅ Filament v4: $view NÃO é static
    protected string $view = 'filament.widgets.selecionar-usuario-agenda-widget';

    public ?int $agendaUserId = null;

    public function mount(): void
    {
        $this->agendaUserId = session('agenda_user_id');

        $this->form->fill([
            'agendaUserId' => $this->agendaUserId,
        ]);
    }

    // ✅ Filament v4: usa Schema (não Form)
    public function form(Schema $form): Schema
    {
        return $form->schema([
            Forms\Components\Select::make('agendaUserId')
                ->label('Selecionar usuário')
                ->options(fn () => User::query()->orderBy('name')->pluck('name', 'id')->all())
                ->searchable()
                ->preload()
                ->live()
                ->afterStateUpdated(function (?int $state) {
                    $this->agendaUserId = $state;

                    session(['agenda_user_id' => $state]);

                    if ($state) {
                        $this->dispatch('agendaUserSelected', userId: $state);
                    }
                }),
        ]);
    }
}
```

3.3) Criar a view do widget

resources/views/filament/widgets/selecionar-usuario-agenda-widget.blade.php

```sh
<x-filament-widgets::widget>
    <x-filament::section>
        {{ $this->form }}
    </x-filament::section>
</x-filament-widgets::widget>
```

4) Atualizar o CalendarWidget para agenda por usuário + escutar o select
4.1) Substitua o arquivo inteiro

```sh
<?php

namespace App\Filament\Widgets;

use App\Models\Evento;
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
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\ValidationException;
use Livewire\Attributes\On;
use Saade\FilamentFullCalendar\Actions;
use Saade\FilamentFullCalendar\Widgets\FullCalendarWidget;

class CalendarWidget extends FullCalendarWidget
{
    use HasWidgetShield;

    public Model|string|null $model = Evento::class;

    public ?int $agendaUserId = null;

    public function mount(): void
    {
        $this->agendaUserId = session('agenda_user_id');
    }

    #[On('agendaUserSelected')]
    public function setAgendaUser(int $userId): void
    {
        $this->agendaUserId = $userId;
        session(['agenda_user_id' => $userId]);

        $this->dispatch('$refresh');
    }

    public static function getHeading(): string
    {
        return 'Calendário';
    }

    public static function canView(): bool
    {
        // ✅ só aparece se usuário foi selecionado
        return (bool) session('agenda_user_id');
    }

    public function config(): array
    {
        return [
            'selectable' => true,
            'unselectAuto' => true,
            'firstDay' => 1,
            'editable' => true,
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

    public function dateClick(): string
    {
        return <<<JS
            function(info) {
                info.view.calendar.el.__livewire.mountAction('create', {
                    start: info.dateStr,
                    end: info.dateStr
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

    private function baseHourOptions(): array
    {
        $hours = array_merge(range(8, 11), range(14, 17));
        $options = [];

        foreach ($hours as $h) {
            $options[sprintf('%02d:00', $h)] = sprintf('%02d:00', $h);
        }

        return $options;
    }

    private function availableHourOptions(?string $dia, ?int $ignoreEventoId = null): array
    {
        $options = $this->baseHourOptions();

        if (! $dia) {
            return $options;
        }

        if (! $this->agendaUserId) {
            return [];
        }

        $ocupados = Evento::query()
            ->where('user_id', $this->agendaUserId)
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

    // ✅ TEM QUE SER PUBLIC (FullCalendarWidget define como public)
    public function getFormSchema(): array
    {
        return [
            Hidden::make('evento_id'),

            TextInput::make('titulo')
                ->label('Título')
                ->required()
                ->maxLength(255),

            Hidden::make('dia')
                ->dehydrated(false),

            Select::make('hora_inicio')
                ->label('Horário')
                ->options(fn (Get $get) => $this->availableHourOptions(
                    $get('dia'),
                    $get('evento_id')
                ))
                ->disabled(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) return false;
                    return empty($this->availableHourOptions($dia, $get('evento_id')));
                })
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) return true;
                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                })
                ->live()
                ->afterStateUpdated(function (?string $state, Set $set, Get $get) {
                    $dia = $get('dia');
                    if (! $dia || ! $state) return;

                    $inicio = Carbon::parse("{$dia} {$state}");
                    $fim = $inicio->copy()->addHour();

                    $set('starts_at', $inicio->toDateTimeString());
                    $set('ends_at', $fim->toDateTimeString());
                }),

            Hidden::make('starts_at')
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) return true;
                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                }),

            Hidden::make('ends_at')
                ->required(function (Get $get) {
                    $dia = $get('dia');
                    if (! $dia) return true;
                    return ! empty($this->availableHourOptions($dia, $get('evento_id')));
                }),

            Placeholder::make('disable_submit_when_no_hours')
                ->label('')
                ->content('')
                ->extraAttributes([
                    'style' => 'display:none',
                    'x-data' => '{}',
                    'x-effect' => <<<'JS'
                        const root =
                            $el.closest('.fi-modal') ||
                            $el.closest('[role="dialog"]') ||
                            document;
                        const submit = root.querySelector('button[type="submit"]');
                        const startsAt = $wire.get('mountedActionsData.0.starts_at');

                        if (submit) {
                            submit.disabled = !startsAt;
                        }
                    JS,
                ]),
        ];
    }

    protected function headerActions(): array
    {
        return [
            Actions\CreateAction::make()
                ->label('Agendar')
                ->mountUsing(function (Schema $form, array $arguments) {
                    if (! $this->agendaUserId) {
                        Notification::make()
                            ->title('Selecione um usuário')
                            ->body('Antes de agendar, selecione um usuário no Dashboard.')
                            ->warning()
                            ->send();

                        $form->fill([
                            'evento_id' => null,
                            'dia' => null,
                            'hora_inicio' => null,
                            'starts_at' => null,
                            'ends_at' => null,
                            'titulo' => null,
                        ]);

                        return;
                    }

                    $dia = isset($arguments['start'])
                        ? Carbon::parse($arguments['start'])->toDateString()
                        : null;

                    if (! $dia) {
                        Notification::make()
                            ->title('Selecione um dia')
                            ->body('Clique em um dia no calendário para agendar.')
                            ->warning()
                            ->send();

                        $form->fill([
                            'evento_id' => null,
                            'dia' => null,
                            'hora_inicio' => null,
                            'starts_at' => null,
                            'ends_at' => null,
                            'titulo' => null,
                        ]);

                        return;
                    }

                    $options = $this->availableHourOptions($dia);

                    if (empty($options)) {
                        $this->notifySemHorario($dia);

                        $form->fill([
                            'evento_id' => null,
                            'dia' => $dia,
                            'hora_inicio' => null,
                            'starts_at' => null,
                            'ends_at' => null,
                            'titulo' => null,
                        ]);

                        return;
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
                        'titulo' => null,
                    ]);
                })
                ->mutateFormDataUsing(function (array $data): array {
                    if (! $this->agendaUserId) {
                        throw ValidationException::withMessages([
                            'titulo' => 'Selecione um usuário para agendar.',
                        ]);
                    }

                    if (empty($data['starts_at'])) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Não há horário disponível para este dia.',
                        ]);
                    }

                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Evento::query()
                        ->where('user_id', $this->agendaUserId)
                        ->whereDate('starts_at', $start->toDateString())
                        ->whereTime('starts_at', $start->format('H:i:s'))
                        ->exists();

                    if ($jaExiste) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Este horário já foi agendado para este usuário. Selecione outro.',
                        ]);
                    }

                    $data['user_id'] = $this->agendaUserId;

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
                    /** @var Evento $record */

                    $startArg = data_get($arguments, 'event.start');
                    $endArg   = data_get($arguments, 'event.end');

                    $start = $startArg
                        ? Carbon::parse($startArg)
                        : Carbon::parse($record->starts_at);

                    $end = $endArg
                        ? Carbon::parse($endArg)
                        : ($record->ends_at
                            ? Carbon::parse($record->ends_at)
                            : $start->copy()->addHour());

                    $dia = $start->toDateString();
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
                    /** @var Evento $record */

                    if (! $this->agendaUserId) {
                        throw ValidationException::withMessages([
                            'titulo' => 'Selecione um usuário para editar.',
                        ]);
                    }

                    $start = Carbon::parse($data['starts_at']);

                    $jaExiste = Evento::query()
                        ->where('user_id', $this->agendaUserId)
                        ->whereKeyNot($record->getKey())
                        ->whereDate('starts_at', $start->toDateString())
                        ->whereTime('starts_at', $start->format('H:i:s'))
                        ->exists();

                    if ($jaExiste) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Este horário já foi agendado para este usuário. Selecione outro.',
                        ]);
                    }

                    unset($data['dia'], $data['hora_inicio'], $data['evento_id']);

                    return $data;
                }),

            Actions\DeleteAction::make(),
        ];
    }

    public function fetchEvents(array $fetchInfo): array
    {
        if (! $this->agendaUserId) {
            return [];
        }

        $start = Carbon::parse($fetchInfo['start'])->startOfDay();
        $end = Carbon::parse($fetchInfo['end'])->endOfDay();

        return Evento::query()
            ->where('user_id', $this->agendaUserId)
            ->where('starts_at', '<', $end)
            ->where(function ($q) use ($start) {
                $q->whereNull('ends_at')->orWhere('ends_at', '>', $start);
            })
            ->get()
            ->map(fn (Evento $e) => [
                'id' => (string) $e->id,
                'title' => $e->titulo,
                'start' => $e->starts_at,
                'end' => $e->ends_at,
            ])
            ->all();
    }
}
```

5) Registrar os widgets no Dashboard (corrigindo namespace)

Abra:

app/Providers/Filament/AdminPanelProvider.php

```sh
->widgets([
    \App\Filament\Widgets\SelecionarUsuarioAgendaWidget::class,
    \App\Filament\Widgets\CalendarWidget::class,
])
```

6) Rodar comandos finais

```sh
composer dump-autoload
php artisan migrate
php artisan filament:clear-cached-components
php artisan optimize:clear
```
