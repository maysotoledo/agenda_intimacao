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

    protected string $view = 'filament.widgets.selecionar-usuario-agenda-widget';

    public ?int $agendaUserId = null;

    public bool $hasEpcUsers = false;

    public bool $hasSingleEpcUser = false;

    public static function canView(): bool
    {
        $user = auth()->user();

        // Se o usuário logado é EPC, ele não escolhe outro usuário.
        return (bool) $user && ! $user->hasRole('epc');
    }

    public function mount(): void
    {
        $epcCount = User::query()->role('epc')->count();

        $this->hasEpcUsers = $epcCount > 0;
        $this->hasSingleEpcUser = $epcCount === 1;

        // Se não há EPC, limpa tudo e mostra placeholder
        if (! $this->hasEpcUsers) {
            session()->forget('agenda_user_id');

            $this->agendaUserId = null;

            $this->form->fill([
                'agendaUserId' => null,
            ]);

            return;
        }

        $sessionUserId = session('agenda_user_id');

        // Valida sessão: só aceita se for EPC
        $validSessionUserId = User::query()
            ->role('epc')
            ->whereKey($sessionUserId)
            ->value('id');

        if ($validSessionUserId) {
            $this->agendaUserId = (int) $validSessionUserId;
        } elseif ($this->hasSingleEpcUser) {
            // Se só existe 1 EPC, auto-seleciona
            $this->agendaUserId = (int) User::query()->role('epc')->value('id');

            session(['agenda_user_id' => $this->agendaUserId]);

            // Dispara evento pra atualizar o calendário imediatamente
            $this->dispatch('agendaUserSelected', userId: $this->agendaUserId);
        } else {
            // Sem seleção válida e há mais de 1 EPC: força escolher
            $this->agendaUserId = null;
            session()->forget('agenda_user_id');
        }

        $this->form->fill([
            'agendaUserId' => $this->agendaUserId,
        ]);
    }

    public function form(Schema $form): Schema
    {
        return $form->schema([
            Forms\Components\Placeholder::make('no_epc_users')
                ->label('')
                ->content('Nenhum usuário com a role "epc" foi encontrado. Crie/atribua essa role a algum usuário para selecionar uma agenda.')
                ->visible(fn (): bool => ! $this->hasEpcUsers),

            Forms\Components\Placeholder::make('single_epc_info')
                ->label('')
                ->content('Existe apenas 1 usuário EPC. A agenda foi selecionada automaticamente.')
                ->visible(fn (): bool => $this->hasSingleEpcUser),

            Forms\Components\Select::make('agendaUserId')
                ->label('Selecionar usuário (EPC)')
                ->placeholder($this->hasSingleEpcUser ? null : 'Selecione um usuário EPC...')
                ->options(fn () => User::query()
                    ->role('epc')
                    ->orderBy('name')
                    ->pluck('name', 'id')
                    ->all()
                )
                ->searchable()
                ->preload()
                ->live()

                // ✅ Filament v4: impede selecionar o placeholder (null) e remove o “clear”
                ->selectablePlaceholder(false)

                ->visible(fn (): bool => $this->hasEpcUsers)
                ->disabled(fn (): bool => $this->hasSingleEpcUser)
                ->required(fn (): bool => $this->hasEpcUsers && ! $this->hasSingleEpcUser)
                ->afterStateUpdated(function (?int $state) {
                    // Proteção extra: se por algum motivo vier null, restaura o valor anterior
                    if (! $state) {
                        if ($this->agendaUserId) {
                            $this->form->fill(['agendaUserId' => $this->agendaUserId]);
                            return;
                        }

                        session()->forget('agenda_user_id');
                        return;
                    }

                    // Garante que o selecionado é EPC
                    $isEpc = User::query()->role('epc')->whereKey($state)->exists();

                    if (! $isEpc) {
                        if ($this->agendaUserId) {
                            $this->form->fill(['agendaUserId' => $this->agendaUserId]);
                        } else {
                            session()->forget('agenda_user_id');
                        }

                        return;
                    }

                    $this->agendaUserId = $state;
                    session(['agenda_user_id' => $state]);

                    $this->dispatch('agendaUserSelected', userId: $state);
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
use App\Models\User;
use BezhanSalleh\FilamentShield\Traits\HasWidgetShield;
use Carbon\Carbon;
use Filament\Actions\Action as FilamentAction;
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
        $user = auth()->user();

        // Se o logado é EPC, sempre mostra a agenda dele
        if ($user?->hasRole('epc')) {
            $this->agendaUserId = (int) $user->getKey();
            session(['agenda_user_id' => $this->agendaUserId]);

            return;
        }

        $sessionUserId = session('agenda_user_id');

        // Valida sessão: só aceita se for EPC
        $validSessionUserId = User::query()
            ->role('epc')
            ->whereKey($sessionUserId)
            ->value('id');

        if ($validSessionUserId) {
            $this->agendaUserId = (int) $validSessionUserId;
            return;
        }

        // Auto-seleciona se só existir 1 EPC
        $epcIds = User::query()->role('epc')->limit(2)->pluck('id');

        if ($epcIds->count() === 1) {
            $this->agendaUserId = (int) $epcIds->first();
            session(['agenda_user_id' => $this->agendaUserId]);
            return;
        }

        // Sem seleção válida
        $this->agendaUserId = null;
        session()->forget('agenda_user_id');
    }

    #[On('agendaUserSelected')]
    public function setAgendaUser(int $userId): void
    {
        // EPC não pode trocar agenda via evento
        if (auth()->user()?->hasRole('epc')) {
            return;
        }

        // Garante que o selecionado é EPC
        $isEpc = User::query()->role('epc')->whereKey($userId)->exists();

        if (! $isEpc) {
            $this->agendaUserId = null;
            session()->forget('agenda_user_id');

            // Atualiza o calendário na tela
            $this->refreshRecords();

            return;
        }

        $this->agendaUserId = $userId;
        session(['agenda_user_id' => $userId]);

        // ✅ Força o FullCalendar a buscar os eventos novamente
        $this->refreshRecords();
    }

    public static function getHeading(): string
    {
        return 'Calendário';
    }

    /**
     * ✅ Importante: não dependa de canView() com session para “aparecer depois”,
     * porque isso não é reativo. Deixe o widget sempre visível e controle por estado.
     */
    public static function canView(): bool
    {
        return auth()->check();
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
            // ✅ Botão para ir ao Dashboard e selecionar o usuário (quando necessário)
            FilamentAction::make('selecionarUsuario')
                ->label('Selecionar usuário')
                ->icon('heroicon-o-user')
                ->visible(fn () => ! auth()->user()?->hasRole('epc') && ! $this->agendaUserId)
                ->url(fn () => route('filament.admin.pages.dashboard')),

            Actions\CreateAction::make()
                ->label('Agendar')
                ->mountUsing(function (Schema $form, array $arguments) {
                    if (! $this->agendaUserId) {
                        Notification::make()
                            ->title('Selecione um usuário')
                            ->body('Clique em "Selecionar usuário" para ir ao Dashboard e escolher um EPC.')
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

1) Criar Model + Migration de Bloqueio
```sh
php artisan make:model Bloqueio -m
```

1.2 Migration (substituir pelo conteúdo abaixo)

Abra o arquivo gerado em database/migrations/*_create_bloqueios_table.php e cole:

```sh
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('bloqueios', function (Blueprint $table) {
            $table->id();

            // EPC bloqueado
            $table->foreignId('user_id')
                ->constrained('users')
                ->cascadeOnDelete();

            // Dia bloqueado (data)
            $table->date('dia');

            // Motivo opcional
            $table->string('motivo')->nullable();

            // Quem criou (admin)
            $table->foreignId('created_by')
                ->nullable()
                ->constrained('users')
                ->nullOnDelete();

            $table->timestamps();

            // Evita duplicidade por EPC + dia
            $table->unique(['user_id', 'dia']);
            $table->index(['user_id', 'dia']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('bloqueios');
    }
};
```

1.3 Model (substituir pelo conteúdo abaixo)

app/Models/Bloqueio.php

```sh
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Bloqueio extends Model
{
    protected $table = 'bloqueios';

    protected $fillable = [
        'user_id',
        'dia',
        'motivo',
        'created_by',
    ];

    protected $casts = [
        'dia' => 'date',
    ];

    protected static function booted(): void
    {
        static::creating(function (Bloqueio $bloqueio) {
            if (empty($bloqueio->created_by)) {
                $bloqueio->created_by = auth()->id();
            }
        });
    }

    public function epc(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function criadoPor(): BelongsTo
    {
        return $this->belongsTo(User::class, 'created_by');
    }
}
```

1.4 Rodar a migration
```sh
php artisan migrate
```

2) Criar Policy (somente admin pode gerenciar bloqueios)

```sh
php artisan make:policy BloqueioPolicy --model=Bloqueio
```

2.2 Policy (substituir o arquivo)

app/Policies/BloqueioPolicy.php:

```sh
<?php

namespace App\Policies;

use App\Models\Bloqueio;
use App\Models\User;

class BloqueioPolicy
{
    private function isAdmin(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'super_admin']);
    }

    public function viewAny(User $user): bool
    {
        return $this->isAdmin($user);
    }

    public function view(User $user, Bloqueio $bloqueio): bool
    {
        return $this->isAdmin($user);
    }

    public function create(User $user): bool
    {
        return $this->isAdmin($user);
    }

    public function update(User $user, Bloqueio $bloqueio): bool
    {
        return $this->isAdmin($user);
    }

    public function delete(User $user, Bloqueio $bloqueio): bool
    {
        return $this->isAdmin($user);
    }
}
```

3) Criar Filament Resource de Bloqueios (Admin)

```sh
php artisan make:filament-resource Bloqueio --simple
```

app/Filament/Resources/Bloqueios/BloqueioResource.php

```sh
<?php

namespace App\Filament\Resources\Bloqueios;

use App\Filament\Resources\Bloqueios\BloqueioResource\Pages\ManageBloqueios;
use App\Models\Bloqueio;
use App\Models\User;
use BackedEnum;
use Carbon\Carbon;
use Filament\Forms\Components\DatePicker;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Resources\Resource;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Schema;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Table;
use Illuminate\Validation\Rule;
use UnitEnum;

class BloqueioResource extends Resource
{
    protected static ?string $model = Bloqueio::class;

    // ✅ Tipagens corretas no Filament v4
    protected static string|BackedEnum|null $navigationIcon = 'heroicon-o-no-symbol';
    protected static string|UnitEnum|null $navigationGroup = 'Agenda';

    protected static ?string $navigationLabel = 'Bloqueios';
    protected static ?string $modelLabel = 'Bloqueio';
    protected static ?string $pluralModelLabel = 'Bloqueios';

    public static function form(Schema $schema): Schema
    {
        return $schema->components([
            Select::make('user_id')
                ->label('EPC')
                ->required()
                ->searchable()
                ->preload()
                ->options(fn () => User::query()
                    ->role('epc')
                    ->orderBy('name')
                    ->pluck('name', 'id')
                    ->all()
                ),

            DatePicker::make('dia')
                ->label('Dia')
                ->required()
                ->native(false)
                ->helperText('Somente dias úteis. Sábado e domingo já são bloqueados automaticamente.')
                ->rules([
                    // ✅ Correção: Filament v4 avalia closures, então ela deve RETORNAR a regra closure
                    fn () => function (string $attribute, $value, \Closure $fail): void {
                        if (! $value) return;

                        $date = Carbon::parse($value);

                        if ($date->isWeekend()) {
                            $fail('Fim de semana já é bloqueado automaticamente. Selecione um dia útil.');
                        }
                    },

                    // ✅ Unicidade: (user_id + dia)
                    fn (Get $get, ?Bloqueio $record) => Rule::unique('bloqueios', 'dia')
                        ->where('user_id', $get('user_id'))
                        ->ignore($record?->getKey()),
                ]),

            TextInput::make('motivo')
                ->label('Motivo (opcional)')
                ->maxLength(255),
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('epc.name')
                    ->label('EPC')
                    ->searchable()
                    ->sortable(),

                TextColumn::make('dia')
                    ->label('Dia')
                    ->date('d/m/Y')
                    ->sortable(),

                TextColumn::make('motivo')
                    ->label('Motivo')
                    ->limit(60),

                TextColumn::make('criadoPor.name')
                    ->label('Criado por')
                    ->toggleable(isToggledHiddenByDefault: true),

                TextColumn::make('created_at')
                    ->label('Criado em')
                    ->dateTime('d/m/Y H:i')
                    ->toggleable(isToggledHiddenByDefault: true),
            ])
            ->defaultSort('dia', 'desc');
    }

    public static function getPages(): array
    {
        return [
            'index' => ManageBloqueios::route('/'),
        ];
    }
}
```

app/Filament/Resources/Bloqueios/BloqueioResource/Pages/ManageBloqueios.php

```sh
<?php

namespace App\Filament\Resources\Bloqueios\BloqueioResource\Pages;

use App\Filament\Resources\Bloqueios\BloqueioResource;
use Filament\Actions\CreateAction;
use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Resources\Pages\ManageRecords;

class ManageBloqueios extends ManageRecords
{
    protected static string $resource = BloqueioResource::class;

    protected function getHeaderActions(): array
    {
        return [
            CreateAction::make(),
        ];
    }

    protected function getRecordActions(): array
    {
        return [
            EditAction::make(),
            DeleteAction::make(),
        ];
    }
}
```

4) Atualizar o Calendário (bloqueios + fim de semana + background vermelho + título desabilitado)

```sh
<?php

namespace App\Filament\Widgets;

use App\Models\Bloqueio;
use App\Models\Evento;
use App\Models\User;
use BezhanSalleh\FilamentShield\Traits\HasWidgetShield;
use Carbon\Carbon;
use Filament\Actions\Action as FilamentAction;
use Filament\Forms\Components\Hidden;
use Filament\Forms\Components\Placeholder;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Utilities\Set;
use Filament\Schemas\Schema;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Collection;
use Illuminate\Validation\ValidationException;
use Livewire\Attributes\On;
use Saade\FilamentFullCalendar\Actions;
use Saade\FilamentFullCalendar\Widgets\FullCalendarWidget;

class CalendarWidget extends FullCalendarWidget
{
    use HasWidgetShield;

    public Model|string|null $model = Evento::class;

    protected static ?int $sort = 2;

    public ?int $agendaUserId = null;

    public function mount(): void
    {
        $user = auth()->user();

        if ($user?->hasRole('epc')) {
            $this->agendaUserId = (int) $user->getKey();
            session(['agenda_user_id' => $this->agendaUserId]);
            return;
        }

        if (! User::query()->role('epc')->exists()) {
            $this->agendaUserId = null;
            session()->forget('agenda_user_id');
            return;
        }

        $sessionUserId = session('agenda_user_id');

        $validSessionUserId = User::query()
            ->role('epc')
            ->whereKey($sessionUserId)
            ->value('id');

        if ($validSessionUserId) {
            $this->agendaUserId = (int) $validSessionUserId;
            return;
        }

        // auto seleciona o primeiro EPC (por nome)
        $firstEpcId = (int) User::query()
            ->role('epc')
            ->orderBy('name')
            ->value('id');

        $this->agendaUserId = $firstEpcId ?: null;

        if ($this->agendaUserId) {
            session(['agenda_user_id' => $this->agendaUserId]);
        } else {
            session()->forget('agenda_user_id');
        }
    }

    #[On('agendaUserSelected')]
    public function setAgendaUser(int $userId): void
    {
        if (auth()->user()?->hasRole('epc')) {
            return;
        }

        $isEpc = User::query()->role('epc')->whereKey($userId)->exists();

        if (! $isEpc) {
            return;
        }

        $this->agendaUserId = $userId;
        session(['agenda_user_id' => $userId]);

        $this->refreshRecords();
    }

    public static function getHeading(): string
    {
        return 'Calendário';
    }

    public static function canView(): bool
    {
        $user = auth()->user();

        if (! $user) {
            return false;
        }

        if ($user->hasRole('epc')) {
            return true;
        }

        return User::query()->role('epc')->exists();
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

    private function isDiaUtil(string $dia): bool
    {
        return ! Carbon::parse($dia)->isWeekend();
    }

    private function isDiaBloqueado(string $dia): bool
    {
        if (! $this->agendaUserId) {
            return false;
        }

        return Bloqueio::query()
            ->where('user_id', $this->agendaUserId)
            ->whereDate('dia', $dia)
            ->exists();
    }

    private function isDiaBloqueadoOuFimDeSemana(string $dia): bool
    {
        return (! $this->isDiaUtil($dia)) || $this->isDiaBloqueado($dia);
    }

    private function notifyDiaBloqueado(string $dia): void
    {
        $data = Carbon::parse($dia)->format('d/m/Y');

        Notification::make()
            ->title('Dia bloqueado')
            ->body("Não é possível agendar em {$data} para este EPC.")
            ->warning()
            ->send();
    }

    private function notifyFimDeSemana(string $dia): void
    {
        $data = Carbon::parse($dia)->format('d/m/Y');

        Notification::make()
            ->title('Fim de semana')
            ->body("Não é possível agendar em {$data}. Agendamentos apenas em dias úteis.")
            ->warning()
            ->send();
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
        if (! $dia) {
            return $this->baseHourOptions();
        }

        if (! $this->agendaUserId) {
            return [];
        }

        if (! $this->isDiaUtil($dia)) {
            return [];
        }

        if ($this->isDiaBloqueado($dia)) {
            return [];
        }

        $options = $this->baseHourOptions();

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

    private function assertDiaAgendavelOrThrow(string $dia): void
    {
        if (! $this->isDiaUtil($dia)) {
            throw ValidationException::withMessages([
                'hora_inicio' => 'Agendamentos apenas em dias úteis (segunda a sexta).',
            ]);
        }

        if ($this->isDiaBloqueado($dia)) {
            throw ValidationException::withMessages([
                'hora_inicio' => 'Este dia está bloqueado para este EPC.',
            ]);
        }
    }

    public function getFormSchema(): array
    {
        return [
            Hidden::make('evento_id'),

            // ✅ Título desabilitado se o dia estiver bloqueado (fim de semana OU bloqueio do admin)
            TextInput::make('titulo')
                ->label('Título')
                ->required()
                ->maxLength(255)
                ->disabled(function (Get $get): bool {
                    $dia = $get('dia');
                    if (! $dia) return false;

                    return $this->isDiaBloqueadoOuFimDeSemana($dia);
                }),

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
            FilamentAction::make('selecionarUsuario')
                ->label('Selecionar usuário')
                ->icon('heroicon-o-user')
                ->visible(fn () => ! auth()->user()?->hasRole('epc') && ! $this->agendaUserId)
                ->url(fn () => route('filament.admin.pages.dashboard')),

            Actions\CreateAction::make()
                ->label('Agendar')
                ->mountUsing(function (Schema $form, array $arguments) {
                    if (! $this->agendaUserId) {
                        Notification::make()
                            ->title('Sem EPC selecionado')
                            ->body('Selecione um EPC para visualizar/agendar.')
                            ->warning()
                            ->send();
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
                        return;
                    }

                    if (! $this->isDiaUtil($dia)) {
                        $this->notifyFimDeSemana($dia);
                        $form->fill(['dia' => $dia, 'hora_inicio' => null, 'starts_at' => null, 'ends_at' => null, 'titulo' => null]);
                        return;
                    }

                    if ($this->isDiaBloqueado($dia)) {
                        $this->notifyDiaBloqueado($dia);
                        $form->fill(['dia' => $dia, 'hora_inicio' => null, 'starts_at' => null, 'ends_at' => null, 'titulo' => null]);
                        return;
                    }

                    $options = $this->availableHourOptions($dia);

                    if (empty($options)) {
                        $this->notifySemHorario($dia);
                        $form->fill(['dia' => $dia, 'hora_inicio' => null, 'starts_at' => null, 'ends_at' => null, 'titulo' => null]);
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
                            'titulo' => 'Selecione um EPC para agendar.',
                        ]);
                    }

                    if (empty($data['starts_at'])) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Não há horário disponível para este dia.',
                        ]);
                    }

                    $start = Carbon::parse($data['starts_at']);
                    $dia = $start->toDateString();

                    $this->assertDiaAgendavelOrThrow($dia);

                    $jaExiste = Evento::query()
                        ->where('user_id', $this->agendaUserId)
                        ->whereDate('starts_at', $dia)
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
                ->mutateFormDataUsing(function (array $data, Model $record): array {
                    if (empty($data['starts_at'])) {
                        throw ValidationException::withMessages([
                            'hora_inicio' => 'Selecione um horário válido.',
                        ]);
                    }

                    $start = Carbon::parse($data['starts_at']);
                    $dia = $start->toDateString();

                    $this->assertDiaAgendavelOrThrow($dia);

                    $jaExiste = Evento::query()
                        ->where('user_id', $this->agendaUserId)
                        ->whereKeyNot($record->getKey())
                        ->whereDate('starts_at', $dia)
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

    private function getBlockedDaysInRange(Carbon $start, Carbon $end): Collection
    {
        if (! $this->agendaUserId) {
            return collect();
        }

        return Bloqueio::query()
            ->where('user_id', $this->agendaUserId)
            ->whereDate('dia', '>=', $start->toDateString())
            ->whereDate('dia', '<=', $end->toDateString())
            ->pluck('dia');
    }

    public function fetchEvents(array $fetchInfo): array
    {
        if (! $this->agendaUserId) {
            return [];
        }

        $rangeStart = Carbon::parse($fetchInfo['start'])->startOfDay();
        $rangeEnd = Carbon::parse($fetchInfo['end'])->endOfDay();

        // eventos normais
        $agendamentos = Evento::query()
            ->where('user_id', $this->agendaUserId)
            ->where('starts_at', '<', $rangeEnd)
            ->where(function ($q) use ($rangeStart) {
                $q->whereNull('ends_at')->orWhere('ends_at', '>', $rangeStart);
            })
            ->get()
            ->map(fn (Evento $e) => [
                'id' => (string) $e->id,
                'title' => $e->titulo,
                'start' => $e->starts_at,
                'end' => $e->ends_at,
            ])
            ->all();

        // bloqueios (admin) em background vermelho + finais de semana em background padrão
        $blockedDays = $this->getBlockedDaysInRange($rangeStart, $rangeEnd)
            ->map(fn ($d) => Carbon::parse($d)->toDateString())
            ->unique()
            ->values();

        $background = [];

        $cursor = $rangeStart->copy();
        while ($cursor->lte($rangeEnd)) {
            $day = $cursor->toDateString();

            if ($cursor->isWeekend()) {
                $background[] = [
                    'id' => 'weekend-' . $day,
                    'start' => $day,
                    'end' => $cursor->copy()->addDay()->toDateString(),
                    'allDay' => true,
                    'display' => 'background',
                ];
            }

            if ($blockedDays->contains($day)) {
                $background[] = [
                    'id' => 'blocked-' . $day,
                    'start' => $day,
                    'end' => $cursor->copy()->addDay()->toDateString(),
                    'allDay' => true,
                    'display' => 'background',
                    // ✅ vermelho
                    'backgroundColor' => 'rgba(255, 0, 0, 0.25)',
                    'borderColor' => 'rgba(255, 0, 0, 0.35)',
                ];
            }

            $cursor->addDay();
        }

        return array_merge($agendamentos, $background);
    }
}
```

5) Limpar caches / otimizações
```sh
php artisan optimize:clear
```
