# fairs-improvements

## -5. Статичный `document.title` и отсутствие динамических заголовков страниц при навигации в Remix (UX/A11y)

- **Страница/Маршрутизация:** Все роуты приложения (`routes/organizations.tsx`, `routes/events.$id.tsx` и др.)
- **Проблема:**
  - При переходах между роутами заголовок вкладки браузера (`document.title`) остается статичным и не отражает текущую страницу.
  - Ухудшается UX при работе с несколькими открытыми вкладками — невозможно визуально найти нужный экран.
  - Нарушается доступность (A11y): ассистивные технологии (скринридеры) не оповещают пользователя с ограниченными возможностями о смене страницы/контекста.

- **Решение:**
  1. Использовать нативную функцию `meta` из Remix в каждом файле роута:

     ```tsx
     import type { MetaFunction } from '@remix-run/node'; // или '@remix-run/react'

     export const meta: MetaFunction = () => {
       return [
         { title: 'Organizations | Admin Portal' },
         { name: 'description', content: 'Manage organizations and accounts' },
       ];
     };
     ```

  2. Для динамических страниц (например, `/events/:id`) использовать данные из `data` (результат работы `loader`):

     ```tsx
     export const meta: MetaFunction<typeof loader> = ({ data }) => {
       if (!data?.event) {
         return [{ title: 'Event Not Found | Admin Portal' }];
       }

       return [
         { title: `${data.event.name} | Admin Portal` },
       ];
     };
     ```

## -4. Антипаттерн «God Hook» (`useEventModel`): агрегация всех запросов/мутаций сущности в один хук

- **Файл/Хук:** `useEventModel.ts`
- **Проблема:**
  1. **Лишние HTTP-запросы и оверхед сети:** Хук-комбайн одновременно вызывает внутри себя 8+ кастомных хуков (`useEventById`, `useEvents`, `useGetEventsWithTicketLevels`, `useEventOptions` и др.). Если компоненту нужна только одна мутация, он всё равно инициализирует все остальные хуки, что приводит к лишним запросам к API.
  2. **Избыточные ререндеры:** Изменение состояния загрузки или данных в *любом* из 8 внутренних хуков провоцирует перерендер компонента-потребителя, даже если эти данные ему не требуются.
  3. **Проблемы с древовидной структурой и Clean Code:** Чтобы отключить ненужные запросы, разработчику приходится передавать полотно флагов (`isEventsEnabled: false`, `isEventOptionsEnabled: false`).
  4. **Усложненная типизация:** Огромные конструкции видов `Parameters<typeof useEvents>[0]['isEnabled']` и `ReturnType<typeof ...>` ухудшают читаемость и замедляют работу TypeScript Server.

- **Решение:**
  - Отказаться от хуков-агрегаторов («моделей»).
  - Следует придерживаться принципа **N+1 (атомарных хуков)**: каждый компонент явно импортирует и вызывает только те хуки, которые ему реально необходимы для работы.
  
  ```tsx
  // ❌ ПЛОХО (God Hook):
  const { createDraftEvent } = useEventModel({
    eventId: id,
    isEventsEnabled: false,
    isEventOptionsEnabled: false,
    isDashboardEventsEnabled: false,
  });

  // ✅ ХОРОШО (Атомарные хуки):
  const { mutate: createDraftEvent } = useCreateDraftEvent();
  const { data: event } = useEventById({ eventId: id });

## -3. Архитектурная монолитность таблицы («God Component») и отсутствие атомарных элементов (UI Pattern)

- **Файл/Компонент:** `TableComponent` / `Table.tsx`
- **Проблема:**
  1. **Нарушение Single Responsibility Principle:** Компонент жестко завязан на TanStack Table (`@tanstack/react-table`), адаптивность, стили отчетов (`reportStyle`), логику слияния заголовков и кастомную печать.
  2. **Огромный оверхед для простых кейсов:** Чтобы отрисовать простую статичную таблицу из двух колонок (например, key-value в модалке или карточке), разработчик вынужден формировать массив `ColumnDef`, мудрить с мета-данными и задействовать тяжеловесный движок TanStack Table.
  3. **Низкая гибкость кастомизации:** Любое нестандартное поведение строки или ячейки требует добавления новых пропсов и ветвлений (`if / else`) в главный компонент.

- **Решение:**
  - Разделить таблицу на **Compound Components** (примитивы): `Table`, `Table.Header`, `Table.Body`, `Table.Row`, `Table.Head`, `Table.Cell`.
  - Оставить эти компоненты чистыми HTML-обёртками со стандартной стилизацией.
  - На основе примитивов построить кастомную обёртку `DataGrid` (или `SmartTable`) для работы с TanStack Table, если нужна сортировка/пагинация/комплексный дата-сетинг.

  **Пример использования атомарного UI (Compound Components):**
  ```tsx
  // Для простых таблиц (2-3 колонки) без TanStack overhead:
  <Table>
    <Table.Header>
      <Table.Row>
        <Table.Head>Param</Table.Head>
        <Table.Head>Value</Table.Head>
      </Table.Row>
    </Table.Header>
    <Table.Body>
      <Table.Row>
        <Table.Cell className="font-bold">Status</Table.Cell>
        <Table.Cell>Active</Table.Cell>
      </Table.Row>
    </Table.Body>
  </Table>

## -2. Пермишены для полей форм: manager vs admin

### Проблема

Правило "поле `pricingClassId` видно только админу" реализовано в двух местах, разными механизмами, без общего источника:

```tsx
// apps/web/src/routes/_manager.events.$id.create.tsx:47
showFeeClassesConfiguration={subdomain === DomainEnum.Admin}   // источник — ПОДДОМЕН
```

```ts
// apps/api/src/modules/ticket-level/dto/admin-ticket-level-create.request.dto.ts
export class AdminTicketLevelCreateRequestDto extends TicketLevelCreateDto {
  pricingClassId!: string;                                     // источник — отдельный DTO + guard по РОЛИ
}
```

Если появится третья роль или поле с другим набором допущенных ролей — придётся руками синхронизировать логику в двух местах на двух языках. Хуже того: фронт ориентируется на **домен**, бэк — на **роль**, а это разные понятия, которые у одного пользователя могут не совпадать.

### Решение — один объект-правило, оба слоя читают его

```ts
// libs/permissions/src/ticket-level.permissions.ts
import { IRoleEnum } from '@fairs/api-core';

export const TicketLevelFieldPermissions = {
  pricingClassId: [IRoleEnum.Admin],
  // maxRedemptions: [IRoleEnum.Admin, IRoleEnum.User], // расширять сюда, а не в JSX и не в DTO
} satisfies Record<string, IRoleEnum[]>;

export const canAccessField = (
  allowedRoles: readonly IRoleEnum[] | undefined,
  role: IRoleEnum,
) => !allowedRoles || allowedRoles.includes(role);
```

**Фронт** — видимость поля решает роль пользователя (из JWT/сессии, не из поддомена):

```tsx
// TicketLevelForm.tsx
const { role } = useCurrentUser();
const showPricingClass = canAccessField(TicketLevelFieldPermissions.pricingClassId, role);

{showPricingClass && <FormItem name="pricingClassId" ... />}
```

**Бэк** — тот же объект решает, принимать ли поле, вместо отдельного DTO-наследования:

```ts
// ticket-level.controller.ts
if (body.pricingClassId && !canAccessField(TicketLevelFieldPermissions.pricingClassId, user.role)) {
  throw new ForbiddenException('pricingClassId is not allowed for this role');
}
```

### Что это даёт

- **Один источник правды**: добавить новое пермишен-поле = одна строка в `TicketLevelFieldPermissions`, а не правка в JSX-условии и в отдельном DTO одновременно.
- **Роль вместо домена**: видимость больше не зависит от того, на каком поддомене открыта страница — только от реальных прав юзера. Поддомен остаётся чисто для выбора layout.
- **Бэк — source of truth по умолчанию**: фронт просто предсказывает то же самое правило для UX (спрятать поле), а бэк всё равно перепроверяет — это не замена валидации, а синхронизация UI с ней.
- **Расширяемость**: для AGENT с `permissionLevel` та же структура расширяется до `Record<string, { roles: IRoleEnum[]; minPermissionLevel?: number }>` без переписывания вызывающего кода.

### Ограничения набросока

Это иллюстрация идеи (~20 строк), не production-ready:
- нет обработки для массового набора DTO/serializer'ов на бэке;
- нет автоматической зачистки запрещённых полей из body до контроллера (сейчас — ручной `if` на каждый эндпоинт).

Возможное следующее развитие: interceptor/pipe на бэке, который чистит запрещённые поля из `body` до контроллера, используя тот же `TicketLevelFieldPermissions`, вместо ручных проверок в каждом методе.


## -1. Проблема: Remix SSR не имеет доступа к правам пользователя → лишний client-side round-trip и мигание UI

### Текущее состояние
- Данные аккаунта и permissions (`accessRules`, `isSuperAdmin`, `socialPermission`, `crmSegmentsVisible`, `crmCampaignsVisible`) грузятся **только на клиенте** через React Query + axios (`useAccountSession`, `useGetMyData` для manager/admin), запрос улетает уже после гидратации.
- Root-лоадер Remix (`apps/web/src/root.tsx`) отдаёт только `ENV`, `theme`, `subdomain` — без аккаунта/прав.
- Кука, которую ставит API, до Remix SSR-запроса **не долетает**: в `apps/web/src/routes/_manager.tsx` эта логика явно закомментирована с TODO *"cookies are not set in calls to Remix BE"*. `session.server.ts` существует, но нигде не импортируется — мёртвый код.
- Из-за этого: (1) первый рендер не знает прав пользователя → мигание/дозагрузка UI после `/me`; (2) `/me` дублируется тремя разными клиентами (public/manager/admin) с разной формой ответа; (3) нет единого контекста с правами — данные растаскиваются пропсами по `_manager.tsx`/`_admin.tsx`.

### Уточнение по tree-shaking (проверено по коду)
Гипотеза «весь код фичей летит всем пользователям независимо от прав» **подтвердилась частично**. Remix уже делает route-based code splitting — CRM-сегменты/кампании лежат в отдельных route-файлах и грузятся отдельным чанком только при переходе на URL. Реальная дыра **уже** — внутри *общих* (shared) роутов: permission-гейтед под-компоненты там импортируются статически (`import`, не `React.lazy`), и их JS подгружается для любого, кто открыл этот роут, — проверка прав отрабатывает уже клиентски, после того как код загружен. Примеры:
- `RenewalViewPage.tsx` (гейт `isSuperAdmin` вокруг `EditOrganization*`)
- `AdminOrganizationEditPage.tsx` (аналогично)
- `_manager.marketing.social.tsx` (`CloudCampaignPage` под гейтом `accessRules`/`socialPermission`)
- `_admin.admin.payouts.tsx`, `_admin.admin.super.tsx` (`isSuperAdmin`)

Lazy-loading (`React.lazy`) в проекте уже используется (`SideBar.Manager`, `SiteChart`, `StatisticsDashboard`, `CheckoutFormStripePayment`), но выбор шёл по размеру бандла/перфу, а не по правам доступа — то есть паттерн есть, просто не применён к permission-гейтам.

### Решение
1. Починить доставку auth-куки/токена до Remix-сервера (сейчас отключено намеренно — нужно разобраться с доменом/поддоменом/SameSite между API и web).
2. В root-лоадере (или лоадерах `_manager`/`_admin`) сходить в API за account + permissions и отдать их через единый контекст (`AccountProvider`/`PermissionsProvider`) вместо трёх разных client-side хуков.
3. В шаренных роутах, где сейчас статический `import` под permission-гейтом (`RenewalViewPage`, `AdminOrganizationEditPage`, `CloudCampaignPage`, `PayoutsPage` и т.п.), заменить его на `React.lazy(() => import(...))`, условие для рендера — уже известные из SSR права, а не после client-side дозапроса.
4. Добавить ревалидацию (`shouldRevalidate`) root/layout-лоадера на login/logout и на смену прав — иначе флаги в контексте залипнут на первом SSR-рендере.

## 0. Рассинхронизация конфигурационных переменных `.env.local` и `.env.example`

- **Файлы:** `.env.local`, `.env.example`
- **Проблема:** Файл `.env.local` значительно отстал от обновленного `.env.example`:
  - **Отсутствуют 70 переменных** (добавленных в `.env.example`), включая ключевые блоки:
    - *Feature Flags:* `FF_CRM_*`, `FF_ADMIN_CALENDAR`, `FF_ADMIN_THEME_SWITCHER`, `FF_ORDER_RESEND_*`, `FF_PAYOUT_*`, `FEATURE_ADMIN_ORDER_DELIVERIES_ENABLED`.
    - *Reach Sync:* `REACH_HISTORICAL_SYNC_*`, `REACH_REALTIME_SYNC_*`, `REACH_SYNC_V2_CONSUMER_ENABLED`, `REACH_DEBUG_ENABLED`, `REACH_DELETE_BATCH_SIZE`.
    - *Sentry & Analytics:* `SENTRY_AUTH_TOKEN`, `SENTRY_CAPTURE_4XX`, `SENTRY_ENABLE_LOGS`, `SENTRY_PINO_BRIDGE_*`, `SENTRY_KIOSK_DSN`, `SENTRY_WEB_EXTRA_SUPPORTED_BROWSERS`, `POSTHOG_FF_API_KEY`, `POSTHOG_KIOSK_*`.
    - *Ticket Lookup:* `TICKET_LOOKUP_SEARCH_RATE_LIMIT_*`, `TICKET_LOOKUP_SESSION_*`, `TICKET_LOOKUP_TURNSTILE_*`, `TICKET_LOOKUP_UPCOMING_ONLY`.
    - *Инфраструктура и таймауты:* `OIDC_ISSUER/JWT_*`, `CLOUD_CAMPAIGN_SSO_*`, `CART_TTL_IN_SECONDS`, `RESERVATION_TTL_MINUTES`, `PDF_RENDERER`, `BULL_MQ_*_CONCURRENCY`, `SLACK_NOTIFICATION_*_CHANNEL_ID`, `APP_MODE`, `MOBILE_TERMINAL_LOCATION_USA/CA` и др.
  - **Присутствуют 19 устаревших/кастомных переменных** в `.env.local` (отсутствующих в `.env.example`):
    - `APPLE_APNS_*`, `APPLE_WALLET_WEB_SERVICE_URL`, `GOOGLE_WALLET_*`, `API_URL_MOBILE`, `ENV_NAME_MOBILE`, `MOBILE_TERMINAL_LOCATION` (старое имя), `SEATS_IO_*`, `TEST_SEATSI`, `STRIPE_FEES_CONNECTED_ACCOUNT`, `USE_STOCK_SERVICE`, `LOADER_CACHE_DISABLED`, `NX_CLI_DISABLE_TUI`.
  - **Мелкие косметические сдвиги:** Инлайн-комментарии съехали на отдельные строки.
- **Решение:**
  1. Актуализировать `.env.local`, подтянув 70 недостающих ключей из `.env.example` с дефолтными значениями (не перезаписывая существующие секреты).
  2. Провести ревизию 19 устаревших ключей (удалить неиспользуемые / переименовать согласно актуальному стандарту).
  3. Добавить скрипт валидации или автоматической проверки окружения (например, через `dotenv-safe` или Zod-схему для env), чтобы сборка выдавала понятную ошибку при отсутствии необходимых переменных.
  

## 1. Использование нативных чекбоксов в фильтрах организаций

- **Страница:** `http://admin.localhost:3000/admin/organizations`
- **Проблема:** В блоке фильтрации используются нативные HTML-чекбоксы (`<input type="checkbox">`), несмотря на наличие кастомного переиспользуемого UI-компонента чекбокса в проекте.
- **Решение:** Заменить нативные чекбоксы на проектный компонент `Checkbox` для соблюдения единого дизайн-кода и консистентности UI.


## 2. Несогласованные стили фокуса элементов UI

- **Компоненты:** Кнопки (`Button`), поля ввода (`Input`), чекбоксы (`Checkbox`)
- **Проблема:** При навигации с клавиатуры (через `Tab`) у интерактивных элементов отображаются разное визуальное оформление состояния фокуса (разные аутлайны, тени или цвета `outline` / `ring`).
- **Решение:** Привести стили фокуса (`:focus-visible`) к единому стандарту в рамках дизайн-системы (единый цвет, толщина и `outline-offset`), обновив базовые стили UI-компонентов или глобальные токены фокуса.


## 3. Проблемы со стилизацией и версткой икон-кнопок действий в списке организаций

- **Страница:** `http://admin.localhost:3000/admin/organizations`
- **Проблемы:**
  - **Семантика:** Первая кнопка действий (иконка перехода/входа) сверстана через `<div>` вместо семантической кнопки (`<button>`) или ссылки (`<a>`), из-за чего она не интерактивна для скринридеров и навигации с клавиатуры.
  - **Цветовая схема:** У иконок действий отсутствует единый цвет (например, вторая иконка синяя, остальные — черные/темные), нет единого визуального гайдлайна для состояния покоя и `:hover`.
  - **Паддинги и фокус:** Внутри элементов задан внутренний отступ (`padding`), из-за чего при клавиатурном фокусе (`:focus-visible`) контур фокуса выглядит криво и неравномерно обрезается или раздувается вокруг иконки.
- **Решение:**
  - Переписать интерактивные элементы на корректные HTML-теги (`<button>` или `<a href="...">`).
  - Вынести эти действия в единый UI-компонент (например, `IconButton`) с фиксированным размером, одинаковыми внутренними отступами (`padding`) и центрированием иконки.
  - Привести иконки к единой палитре цветов и одинаковому стилю состояния фокуса/наведения.
<img width="1829" height="931" alt="image" src="https://github.com/user-attachments/assets/211fcdab-1f7d-4047-96f7-0e7fce4b2f1e" />
<img width="1735" height="609" alt="image" src="https://github.com/user-attachments/assets/df8e4914-b27e-44cc-96f1-daa890db8783" />


## 4. Кастомная верстка таблицы организаций на `<div>` вместо компонента `Table`

- **Страница:** `http://admin.localhost:3000/admin/organizations`
- **Проблема:** Таблица с перечнем организаций сверстана вручную с использованием тегов `<div>` и кастомных CSS-классов вместо применения готового переиспользуемого компонента `Table` / `DataTable`. Из-за этого теряется семантика таблицы, ухудшается доступность (a11y) и нарушается единообразие табличных данных в проекте.
- **Решение:** Рефакторинг страницы с переводом таблицы на проектный UI-компонент `Table` для сохранения единой структуры, стилей и удобной поддержки.


## 5. Избыточное приведение типов для `userData`

- **Страница/Файл:** Компонент списка организаций (`[http://admin.localhost:3000/admin/organizations](http://admin.localhost:3000/admin/organizations)`)
- **Проблема:** В коде используется явное приведение типа (type assertion) через `as`:
  ```typescript
  const isSuperAdmin = Boolean(
    (userData as { isSuperAdmin?: boolean } | undefined)?.isSuperAdmin,
  );
  ```
  Это утверждение типов избыточно, так как объект `userData` уже содержит поле `isSuperAdmin` в своем базовом типе.
- **Решение:** Убрать лишнее приведение типов и обращаться к свойству напрямую:
  ```typescript
  const isSuperAdmin = Boolean(userData?.isSuperAdmin);
  ```


## 6. Избыточный `useCallback` для обработчика ввода поиска

- **Страница/Файл:** Компонент списка организаций (`http://admin.localhost:3000/admin/organizations`)
- **Проблема:** Функция `handleSearchChange` обёрнута в `useCallback`:
  ```typescript
  const [search, setSearch] = useState('');
  const handleSearchChange = useCallback(
    (event: React.ChangeEvent<HTMLInputElement>) => {
      setSearch(event.target.value);
    },
    [],
  );
  ```
  Это избыточная оптимизация, так как:
  1. `SearchComponent` не обёрнут в `React.memo` и перерисовывается при каждом рендере родителя.
  2. При вводе символа меняется `search` в `useState`, из-за чего в `SearchComponent` передаётся новый проп `value`, с провоцируя его ререндер в любом случае.
- **Решение:** Удалить `useCallback` и объявить обычную функцию или передавать inline-инлайн обработчик:
  ```typescript
  const [search, setSearch] = useState('');
  const handleSearchChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setSearch(event.target.value);
  };
  ```


## 7. Дублирование типа `Event` между страницами и компонентами

- **Страница/Файлы:** `OrganizationsListPage` и `OrganizationsList`
- **Проблема:** В обоих файлах продублировано абсолютно одинаковое объявление типа `Event`:
  ```typescript
  export type Event = {
    id: string;
    name: string;
    description: string;
    status: EventStatus;
    timeStart?: string | Date;
    timeEnd?: string | Date;
    timezone?: string;
    coverPhotoUrl?: string;
    createdAt: string;
    updatedAt: string;
  };
  ```
- **Решение:** Вынести тип `Event` в общий модуль типов (или файл `types.ts` текущей фичи) и импортировать его в компонентах, исключив дублирование:
  ```typescript
  import type { Event } from './types';
  ```


## 8. Оптимизация компонента `PageTitle`

- **Файл:** Компонент `PageTitle`
- **Код:**
  ```tsx
  export const PageTitle = ({
    children,
    neutral,
    className = '',
  }: {
    children: ReactNode;
    neutral?: boolean;
    className?: string;
  }) => (
    <h2
      role="heading"
      aria-level={2}
      className={`${neutral? 'mb-3 text-lg text-neutral-500 md:text-xl' : 'text-xl font-bold md:text-4xl'} ${className}`}>
      {children}
    </h2>
  );
  ```
- **Проблемы:**
  1. **Лишние ARIA-атрибуты:** Тег `<h2>` семантически уже имеет роль `heading` и уровень `2`. Атрибуты `role="heading"` и `aria-level={2}` избыточны.
  2. **Склейка CSS-классов:** Ручное объединение Tailwind-классов через шаблонные строки трудно читается и может вызывать конфликты с внешним `className`.
  3. **Типизация `children`:** Вместо ручного описания `children: ReactNode` предпочтительнее использовать встроенный тип `PropsWithChildren`.
- **Решение:** Убрать лишние атрибуты и применить утилиту слияния классов (`clsx` или `cn`):
  ```tsx
  import { FC, PropsWithChildren } from 'react';
  import { cn } from '@/lib/utils'; // или clsx

  type PageTitleProps = PropsWithChildren<{
    neutral?: boolean;
    className?: string;
  }>;

  export const PageTitle: FC<PageTitleProps> = ({ children, neutral, className }) => (
    <h2
      className={cn(
        neutral
          ? 'mb-3 text-lg text-neutral-500 md:text-xl'
          : 'text-xl font-bold md:text-4xl',
        className,
      )}>
      {children}
    </h2>
  );
  ```


## 9. Неоптимальная стратегия ключей React Query в хуке `useGetMeData`

- **Файл:** `useGetMeData` (или кастомный запрос пользователя)
- **Код:**
  ```typescript
  const keys = useMemo(() => ['useGetMyUserQuery'], []);
  ```
- **Проблемы:**
  1. **Избыточный `useMemo`:** Мемоизация статичного массива без зависимостей внутри хука создаёт лишние накладные расходы.
  2. **Сложность инвалидации:** Ключ объявлен локально внутри хука, из-за чего его невозможно импортировать в мутациях или других местах для вызова `queryClient.invalidateQueries(...)` или `setQueryData`.
- **Решение:** Вынести `queryKey` за пределы хука в виде экспортируемой константы:
  ```typescript
  export const GET_ME_QUERY_KEY = ['useGetMyUserQuery'] as const;

  export const useGetMeData = ({ isEnabled }: UseGetMyUserProps): UseGetMyUserResponse => {
    const { data, isLoading, isSuccess } = useQueryBuilder<IUserResponseDto>({
      callback: () => adminClient.api.authControllerMe(),
      queryKey: GET_ME_QUERY_KEY,
      enabled: isEnabled,
      // ...
    });
    // ...
  };
  ```


## 10. Недружелюбное сообщение об ошибке при авторизации (UX/UI)

- **Страница:** Авторизация (`/login`)
- **Проблема:** При вводе неверных учетных данных система выводит сырую техническую ошибку axios/fetch:
  > `Request failed with status code 401`
  
  Пользователю отдаются технические детали (код статуса HTTP), вместо понятного бизнес-сообщения.
- **Решение:** 
  1. Перехватывать ошибки авторизации (401 Unauthorized) на уровне API-клиента или формы.
  2. Заменить сообщение на понятный текст, например: *"Invalid email or password. Please try again."* (или *"Неверный email или пароль"* в зависимости от локали).

<img width="598" height="703" alt="image" src="https://github.com/user-attachments/assets/6c1968c6-7a5c-4ceb-a40b-f9ff4b84f3cf" />


## 11. Смещение текста валидационной ошибки поверх соседних элементов формы

- **Форма:** Создание/редактирование организации (`Organization name`)
- **Проблема:** При появлении сообщения об ошибке валидации (например, *"Organization name is required."*) текст ошибки накладывается поверх нижестоящего поля ввода (`Description`). Это происходит из-за некорректного использования `position: absolute` у блока ошибки без фиксированного контейнера либо из-за отсутствия отрицательного `margin` / отступа у контейнера поля.
- **Решение:**
  - Убрать абсолютное позиционирование у текста ошибки (`FieldError` / `FormMessage`) и выводить его в общем потоке документа (`static` / `relative`).
  - Либо зарезервировать фиксированную высоту под текст ошибки в обертке поля (FormItem / InputGroup), чтобы появление текста ошибки не ломало верстку и не перекрывало соседей.
<img width="883" height="100" alt="image" src="https://github.com/user-attachments/assets/67ca612f-1de0-446e-af71-9ec189f1c5bc" />



## 12. Непоследовательное использование обертки `FormItem` на странице `NewOrganizationPage`

- **Страница/Файл:** `NewOrganizationPage`
- **Проблема:** В рамках одной формы используется смешанный подход к обертке и лейблингу полей ввода:
  - Часть полей использует кастомный UI-компонент обертки `FormItem`:
    ```tsx
    <FormItem required title="Organization name">
      <FormInput control="{control}" helperText="{errors.name?.message}" isError="{!!errors.name}" name="name" placeholder="Company name" rules="{createOrganizationFormRules.name}"/>
    </FormItem>
    ```
  - Другая часть полей сверстана через нативные HTML-теги `<div>` и `<p>`:
    ```tsx
    <div>
      <p className="text-sm text-neutral-400">Address</p>
      <div className="w-full">
        <FormInput<CreateOrganizationFormPayload>
          control={control}
          name="generalAddress"
          placeholder="e.g., 123 Main St"
        />
      </div>
    </div>
    ```
  Это приводит к рассинхрону в визуальных отступах, стилях лейблов (`text-neutral-400` vs стили `FormItem`), обработке валидационных ошибок и звёздочек обязательных полей (`required`).

- **Решение:**
  1. Привести все элементы формы на странице к единому стандарту с использованием компонента `FormItem`.
  2. Унифицировать связку `FormItem` + `FormInput` в единый дизайн-код (или заложить пропы `label`/`required`/`error` напрямую в `FormInput`, если `FormItem` всегда используется как его родитель), чтобы исключить ручную верстку оберток полей.



## 13. Декомпозиция формы и оптимизация передачи контекста на `NewOrganizationPage`

- **Страница/Файл:** `NewOrganizationPage`
- **Проблемы:**
  1. **Монолитность формы:** Форма содержит 4 больших блока с множеством полей в одном файле. Читать, поддерживать и изолировать логику отдельных секций в таком виде тяжело.
  2. **Проп-дриллинг `control`:** Во все компоненты полей ввода ручным образом прокидывается проп `control` из `useForm` (`control={control}`).
  3. **Семантика группировки:** Секции формы не сгруппированы семантически для скринридеров и вспомогательных технологий.
- **Решение:**
  1. Обернуть форму в `FormProvider` из `react-hook-form`. Это позволит дочерним инпутам получать `control` и состояние ошибок автоматически через хук `useFormContext()`, не засоряя пропсы.
  2. Вынести логические секции формы в отдельные подкомпоненты (например, `GeneralInfoSection`, `AddressSection` и т.д.).
  3. Использовать семантические теги `<fieldset>` и `<legend>` для каждого логического блока:
     ```tsx
     <fieldset className="space-y-4 rounded-lg border p-4">
       <legend className="px-2 text-lg font-semibold">Address Information</legend>
       {/* Секция с полями */}
     </fieldset>
     ```


## 14. Избыточные `useEffect` для обработки логики сброса и синхронизации полей формы

- **Страница/Файл:** `NewOrganizationPage`
- **Код:**
  ```typescript
  const onChangeWebsiteSelector = useCallback(() => {
    resetField('dudaWebsiteName');
  }, [resetField]);

  useEffect(() => {
    setValue('state', undefined);
    if (isTaxationEnabled === TaxationEnabledOptionsValues.DEFAULT) {
      setValue('taxAddressState', null);
    }
  }, [country, isTaxationEnabled, setValue]);

  const onChangeCountryListener = (payload?: string) => {
    setValue('timezone', timeZonesOptions(payload as ICountry)[0]?.value || '');
  };

  useEffect(() => {
    if (isTaxationEnabled === TaxationEnabledOptionsValues.NON_TAXABLE) {
      setValue('taxAddressState', null);
      setValue('taxAddressCity', null);
      setValue('taxAddressPostalCode', null);
      setValue('taxAddressLine1', null);
      setValue('taxAddressLine2', null);
    }
  }, [isTaxationEnabled, setValue]);
  ```
- **Проблемы:**
  1. **Антипаттерн синхронизации состояния через `useEffect`:** Побочные эффекты реагируют на изменения значений формы в циклах эффектов, вызывая каскадные лишние ререндеры компонента.
  2. **Сложность отслеживания связей:** Логика сброса зависимых полей (`taxAddressState`, `timezone`, `dudaWebsiteName`) размазана по разным эффектам и слушателям, вместо того чтобы находиться в месте наступления самого события.
- **Решение:**
  - Полностью отказаться от `useEffect` для управления состоянием полей формы.
  - Перенести всю зависимую логику (сброс адреса, установка таймзоны) в явные обработчики событий (`onChange` соответствующих селекторов/радио-кнопок):
    ```typescript
    const handleTaxationChange = (value: TaxationEnabledOptionsValues) => {
      setValue('isTaxationEnabled', value);
      if (value === TaxationEnabledOptionsValues.NON_TAXABLE) {
        setValue('taxAddressState', null);
        setValue('taxAddressCity', null);
        setValue('taxAddressPostalCode', null);
        setValue('taxAddressLine1', null);
        setValue('taxAddressLine2', null);
      } else if (value === TaxationEnabledOptionsValues.DEFAULT) {
        setValue('taxAddressState', null);
      }
    };
    ```


## 15. Непоследовательная структура файлов опций и селектов (`selectableValues.ts` vs `websiteStatusOptions.ts`)

- **Папка:** `NewOrganizationPage/config/`
- **Файлы:** `selectableValues.ts`, `websiteStatusOptions.ts`
- **Проблема:** В папке конфигураций нарушена единообразная структура экспорта списков опций для селекторов (`SelectOption`):
  - В файле `selectableValues.ts` сгруппировано несколько массивов с опциями (например, `taxationEnabledOptions` и др.).
  - В отдельном файле `websiteStatusOptions.ts` лежит всего один массив `websiteStatusOptions`.
  
  Такой подход создает хаос в импортах и файловой структуре — нет четкого правила, создается ли под каждый селект отдельный файл или все опции формы живут в едином модуле.

- **Решение:**
  - Привести структуру файлов к единому стандарту в рамках модуля/страницы:
    - **Вариант А (рекомендуемый):** Объединить все опции селекторов формы в один общий файл конфигурации (например, `organizationFormOptions.ts` или `selectableValues.ts`).
    - **Вариант Б:** Если опции содержат сложную логику или выносятся отдельно, разбить абсолютно все опции на отдельные модульные файлы (`taxationOptions.ts`, `websiteStatusOptions.ts`, `timezoneOptions.ts` и т.д.).


## 16. Отсутствие единого стандарта объявления функций (Function Declarations vs Arrow Functions)

- **Контекст:** Весь проект / Стандарты написания кода
- **Проблема:** В проекте используется смешанный подход: часть компонентов и утилит объявлена через `function name() {}`, а часть — через `const name = () => {}`. Это создает визуальный рассинхрон и разнобой в стиле кода.
- **Решение:**
  1. Зафиксировать правило в кодовой базе (и внести в гайдлайн/линтер): **по умолчанию всегда использовать стрелочные функции (`arrow functions`)**.
  2. Использовать `function declaration` только в исключительных случаях:
     - Если критически необходим `hoisting` (поднятие функции выше по коду).
     - Если требуется динамический доступ к `this` / `arguments` (например, в некоторых специфических хелперах или классическом ООП-коде).
  3. Настроить ESLint-правило `func-style` для автоматического контроля:
     ```json
     "func-style": ["error", "declaration", { "allowArrowFunctions": true }]
     ```


## 17. Отсутствие мемоизации вычисляемых опций таймзон (`timeZonesOptions`)

- **Страница/Файл:** `NewOrganizationPage`
- **Код:**
  ```typescript
  const tzOptions = timeZonesOptions(country);
  ```
- **Проблема:**
  1. При каждом рендере формы (например, при каждом вводе символа в любое текстовое поле) заново вызывается `.map()` по массиву таймзон страны.
  2. Генерируется новая ссылка на массив `tzOptions`, из-за чего кастомный компонент выбора (`Select`) повторно перерисовывается при любых изменениях состояния родительской формы.
- **Решение:** Обернуть вычисление опций в `useMemo` с зависимостью от выбранной страны (`country`):
  ```typescript
  const tzOptions = useMemo(() => timeZonesOptions(country), [country]);
  ```


## 18. Использование TypeScript `enum` вместо `as const` объектов (Const Assertions)

- **Контекст:** Проект / Типизация констант
- **Код:**
  ```typescript
  export enum FeeAbsorptionOptionValues {
    PASS = 'pass',
    ABSORB = 'absorb',
  }
  ```
- **Проблема:** 
  1. TypeScript `enum` создаёт лишний JS-код при сборке (IIFE-функцию в бандле), увеличивая размер приложения.
  2. Требует явного импорта самóго `enum` в местах вызова, не позволяя передавать обычный строковый литерал (например, `'pass'`), даже если его значение совпадает.
- **Решение:** Перейти на стандартную паттерн-структуру с использованием `as const` и генерацией типа через `typeof`:
  ```typescript
  export const FeeAbsorptionOptionValues = {
    PASS: 'pass',
    ABSORB: 'absorb',
  } as const;

  export type FeeAbsorptionOptionValue =
    typeof FeeAbsorptionOptionValues[keyof typeof FeeAbsorptionOptionValues];
  ```

## 19. Наличие неиспользуемого кода и отсутствие жестких ESLint-правил автофикса

- **Контекст:** Весь проект / Конфигурация линтера (`.eslintrc` / `eslint.config.js`)
- **Проблема:** В файлах проекта остаются неиспользуемые переменные, импорты и аргументы функций, а линтер либо молчит, либо выдает слабое предупреждение (`warning`), не блокируя сборку и коммит.
- **Решение:**
  1. Перевести правило неиспользуемых переменных в статус ошибки (`error`).
  2. Подключить плагин `eslint-plugin-unused-imports`, который умеет автоматически вычищать лишние импорты и переменные при сохранении файла или запускe `eslint --fix`.
  3. Добавить в `.eslintrc`:
     ```json
     {
       "plugins": ["unused-imports"],
       "rules": {
         "no-unused-vars": "off",
         "@typescript-eslint/no-unused-vars": "off",
         "unused-imports/no-unused-imports": "error",
         "unused-imports/no-unused-vars": [
           "error",
           {
             "vars": "all",
             "varsIgnorePattern": "^_",
             "args": "after-used",
             "argsIgnorePattern": "^_"
           }
         ]
       }
     }
     ```
  4. Настроить Husky / `lint-staged`, чтобы перед каждым коммитом неиспользуемый код автоматически вычищался и не попадал в репозиторий.


## 20. Использование `lodash/uniqueId` вместо нативного хука `useId` (React 18+) для доступности форм

- **Контекст:** Компоненты форм (`FormInput`, `FormItem`, кастомные инпуты)
- **Проблема:** 
  1. Для генерации связки `htmlFor` у `<label>` и `id` у `<input>` используется `lodash/uniqueId`. На всём фронтенде этот вызов встречается **всего 4 раза**.
  2. Это вызывает ошибки гидратации (**Hydration Mismatch**) при серверном рендеринге (SSR), так как счетчики `uniqueId` на сервере и клиенте расходятся.
  3. Проект тащит ненужную зависимость/модуль ради функции, под которую в React 18 уже есть встроенный механизм.
- **Решение:**
  - Заменить все 4 вызова `lodash/uniqueId` на нативный хук React 18 — `useId()`.
  - Удалить `lodash` / `lodash.uniqueid` из `package.json` и избавиться от лишнего модуля в бандле.
  - Пример использования:
    ```tsx
    import { useId } from 'react';

    export const InputWithLabel = ({ label, ...props }) => {
      const id = useId();

      return (
        <div>
          <label htmlFor={id}>{label}</label>
          <input id={id} {...props} />
        </div>
      );
    };
    ```


## 21. Оверхед, лишние абстракции и ручная переупаковка результатов в `useCreateOrganization`

- **Файл/Хук:** `useCreateOrganization.ts`
- **Проблемы:**
  1. **Избыточная деструктуризация и проброс полей:** Хук вручную достаёт `mutateAsync`, `isPending`, `isSuccess`, `isError`, переименовывает их в `create`/`isLoading` и собирает заново в кастомный объект. Это лишний слой абстракции, создающий дублирующий бойлерплейт.
  2. **Лишний `useCallback`:** Функция `create` просто пробрасывает вызов в `mutateAsync`. Посколько `mutateAsync` из React Query имеет стабильную ссылку, дополнительный `useCallback` бессмыслен.
  3. **Избыточный `useMemo` для ключа:** Мемоизация статического массива `['useCreateOrganizationMutation']` не даёт никакой пользы.
  4. **Избыточный интерфейс возвращаемого значения:** Тип `UseCreateOrganizationResponse` дублирует сигнатуру хука, вместо использования automatic type inference.
  5. **Смешение ответственности:** Сложная логика маппинга и маскирования данных из `CreateOrganizationFormPayload` в DTO бэкенда написана inline прямо внутри колбэка мутации.
  6. **Опасное приведение типов:** Использование `payload.feeRules!` (non-null assertion operator) и небезопасные преобразования типов вроде `!!Number(payload.isEnabledWebsite)`.

- **Решение:**
  - Вынести трансформацию данных формы в DTO в отдельную чистую функцию `mapFormToCreateOrganizationDto`.
  - Убрать `useMemo`, `useCallback` и явный тип ответа `UseCreateOrganizationResponse`.
  - Возвращать результат `useMutationBuilder` напрямую без ручной распаковки и запаковки.
  - Упрощенный вид хука:

  ```typescript
  export const useCreateOrganization = () => {
    return useMutationBuilder({
      mutationKey: ['useCreateOrganizationMutation'],
      callback: (payload: CreateOrganizationFormPayload) =>
        adminClient.api.adminOrganizationControllerCreate({
          iOrganizationCreateRequestDto: mapFormToCreateOrganizationDto(payload),
        }),
    });
  };
  ```


## 22. Использование `Select` вместо `Switch` / `Checkbox` для бинарных опций (UX)

- **Компонент/Форма:** Поля со статусами (например, `isEnabledWebsite`, `isTaxationEnabled`)
- **Проблема:** Для выбора между двумя взаимоисключающими состояниями (`Enabled` / `Disabled`) используется выпадающий список (`Select`). Это создает лишний оверхед в интерфейсе:
  1. Требует от пользователя 2 кликов вместо 1.
  2. Скрывает доступные опции внутри дропдауна.
  3. Не соответствует общепринятым гайдлайнам UI/UX (NN/g, Material Design, Human Interface Guidelines).
- **Решение:** 
  - Заменить элементы выпадающего списка `Select` с бинарным выбором на переключатель `Switch` (`Toggle`) или `Checkbox`.
  - В форме передавать чистый `boolean` (`true` / `false`), избавившись от искусственных строковых значений (`'1'` / `'0'` или `'enabled'` / `'disabled'`).
<img width="287" height="168" alt="image" src="https://github.com/user-attachments/assets/5c4f0c67-3d03-4c0d-958f-332e02469016" />



## 23. Отсутствие кнопки показа/скрытия пароля в полях ввода (UX)

- **Страница/Компоненты:** Авторизация (`/login`), регистрация и любые формы с `FormInput type="password"`
- **Проблема:** Поля ввода пароля не имеют функции переключения видимости (toggle password visibility).
  - Пользователь не может визуально проверить корректность введенных символов, маскированных точками (`••••••••`).
  - При случайной ошибке ввода приходится полностью стирать и перебирать весь пароль заново.
  - Повышается количество невалидных попыток входа и технических ошибок 401.
- **Решение:**
  1. Добавить встроенную кнопку-иконку («глаз») в правый край компонента `FormInput` для полей типа `password`.
  2. Реализовать локальное состояние переключения типа инпута между `type="password"` и `type="text"`.
  3. Обеспечить доступность (a11y): добавлять понятный `aria-label` (например, *"Show password"* / *"Hide password"*).


## 24. Фиксированная высота `textarea` без авторазворачивания при вводе текста (UX)

- **Компоненты/Формы:** Все компоненты `Textarea` / `FormTextarea` (поля описания, комментариев и т.д.)
- **Проблема:** Текстовое поле имеет статичную высоту и не адаптируется под объем введенного содержимого.
  - При вводе длинного текста появляется внутренний скроллбар, скрывая часть текста из поля зрения.
  - Пользователю неудобно вычитывать и редактировать длинные описания, приходится постоянно скроллить внутри маленького блока.
- **Решение:**
  - Добавить автоматическую подстройку высоты компонента под контент.
  - В современных браузерах использовать нативное CSS-свойство:
    ```css
    textarea {
      field-sizing: content;
    }
    ```
  - В качестве фолбэка / кроссбраузерного решения использовать библиотеку `react-textarea-autosize` или кастомный хук расчета высоты по `scrollHeight`.


## 25. Отсутствие `debounce` в инпутах поиска (Критическая нагрузка на API и Race Conditions)

- **Страница/Компоненты:** Таблица организаций (`Organizations`), все поисковые инпуты и фильтры по проекту
- **Проблема:**
  1. На каждый введенный символ в поисковой строке мгновенно отправляется отдельный HTTP-запрос к API (Network tab заливает десятки одинаковых `GET /organizations?...` на каждую клавишу).
  2. Создает огромную избыточную нагрузку на сервер и БД (по сути, самодельный DDoS своего же бэкенда).
  3. Появляется высокий риск **Race Condition** (состояние гонки), когда более медленный ответ на ранний символ перекрывает актуальный результат поиска.

- **Решение:**
  1. Внедрить задержку отправки запросов (отправка через 300–500 мс после завершения ввода).
  2. Посколько в проекте уже есть `lodash.debounce`, использовать его, но обязательно обернуть в `useCallback` (или `useRef`), чтобы функция не пересоздавалась на каждый рендер компонентов React:

     ```tsx
     import { useCallback, useEffect, useState } from 'react';
     import debounce from 'lodash/debounce';

     export const SearchInput = ({ onSearch }: { onSearch: (val: string) => void }) => {
       const [value, setValue] = useState('');

       // Сохраняем стабильную ссылку на debounced-функцию
       const debouncedSearch = useCallback(
         debounce((searchQuery: string) => {
           onSearch(searchQuery);
         }, 300),
         [onSearch]
       );

       // Очищаем таймер при размонтировании компонента
       useEffect(() => {
         return () => {
           debouncedSearch.cancel();
         };
       }, [debouncedSearch]);

       const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
         const val = e.target.value;
         setValue(val);
         debouncedSearch(val);
       };

       return (
         <input
           type="text"
           value={value}
           onChange={handleChange}
           placeholder="Search..."
         />
       );
     };
     ```
  3. Провести аудит всех поисковых строк, полей фильтрации и автокомплитов по всему проекту на наличие задержки ввода.
<img width="2254" height="1073" alt="image" src="https://github.com/user-attachments/assets/6ed64889-dc03-4680-82ec-e39f2e354ba3" />



## 26. Невалидная вложенность интерактивных элементов (`<a>` внутри `<button>`) и отсутствие полиморфизма компонента `Button`

- **Контекст:** Компоненты UI / Ссылки-кнопки
- **Код:**
  ```tsx
  <a href={linkToPublicEvent} target="_blank" rel="noreferrer">
    <Button icon="{<EyeIcon"/>}
      label="View listing"
      kind="secondary"
      size="md"
    />
  </a>
  ```
- **Проблема:**
  1. Нарушение спецификации HTML: вкладывать `<button>` внутрь `<a>` запрещено стандартом (Interactive content nesting).
  2. Сломанная доступность (a11y): скринридеры воспринимают конструкт как два вложенных интерактивных элемента.
  3. Риск непредсказуемого всплытия событий (event bubbling) и двойных кликов.

- **Решение:**
  - Поддержать в компоненте `Button` полиморфный рендеринг через проп `as` или передачу пропа `href` (рендерить `<a>` со стилями кнопки, если передан `href` / `as="a"`):
  
  ```tsx
  // Вариант А: Использование свойства as / href у Button
  <Button as="a" href="{linkToPublicEvent}" icon="{<EyeIcon" rel="noreferrer" target="_blank"/>}
    label="View listing"
    kind="secondary"
    size="md"
  />
  ```

  - В крайнем случае (если это переход внутри SPA) использовать одиночную `<Button onClick={...} />` с программным навигатором, либо оформить ссылку тегом `<a>` с классом/стилями кнопки.



## 27. Переусложнение типов, мертвый код и полное игнорирование a11y в компоненте `TabHeaders`

- **Файл/Компонент:** `TabHeaders.tsx`
- **Проблемы:**
  1. **Мёртвые пропсы и оверхед типизации:** Сложный Discriminated Union (`ExclusiveTabHeadersProps`) для `useHashChange` и `useQueryParams` создаёт огромную кодовую базу типов, но сами эти пропсы **абсолютно не используются** в логике компонента.
  2. **Сломанная и странная логика `onClick`:**
     - При отсутствии `selectedTab` клик блокируется и выводится `console.warn`, вместо того чтобы вызвать `onChange(tab.value)`.
     - Наличие пустого ветвления `if (selectedTab === tab.value) {}`.
  3. **Замазывание a11y через `eslint-disable`:** Табы свёрстаны на `<div>` вместо `<button>`, отсутствуют атрибуты `aria-selected`, `tabIndex`, а также обработка клавиш. Вместо исправления доступности использованы заглушки `eslint-disable-next-line jsx-a11y/*`.
  4. **Плохие ключи списка (`key`):** `key={`${tab.value}-${tab.label}`}` — избыточная конкатенация строк, когда `tab.value` уже является уникальным идентификатором.

- **Решение:**
  - Убрать мёртвую логику `useHashChange` / `useQueryParams` из базового компонента.
  - Использовать семантичные элементы `<button>` с корректными атрибутами accessibility (`aria-selected`, `role="tab"`).
  - Упростить код и сделать его чистым презентационным компонентом.



## 28. Отсутствие явных визуальных стилей для состояния `disabled` у чекбоксов (UI/UX)

- **Страница/Компоненты:** Формы с чекбоксами (например, `Select Features to Enable`)
- **Проблема:** Заблокированные чекбоксы визуально никак не отличаются от активных (одинаковый цвет границы, лейбла и фона).
  - Пользователю приходится наводить курсор на каждый чекбокс, чтобы понять по `cursor: not-allowed`, доступен ли элемент для взаимодействия.
  - Нарушаются гайдлайны visual feedback (пользователь ожидает мгновенно считывать недоступные опции).
- **Решение:**
  - Добавить в CSS/Tailwind стили для состояния `:disabled` / `disabled`:
    - Снижать прозрачность всего блока: `opacity-50` или `opacity-60`.
    - Применять серый цвет для текста лейбла (`text-neutral-400`).
  
  ```tsx
  <label
    className={`flex items-center gap-2 ${
      isDisabled ? 'cursor-not-allowed opacity-50' : 'cursor-pointer'
    }`}
  >
    <input
      type="checkbox"
      disabled={isDisabled}
      className="disabled:bg-neutral-100 disabled:border-neutral-300 disabled:cursor-not-allowed"
    />
    <span className={isDisabled ? 'text-neutral-400' : 'text-neutral-900'}>
      {label}
    </span>
  </label>



## 29. Скачки интерфейса (Layout Shift / CLS) при переключении фильтров периода на графике

- **Страница/Компоненты:** Аналитика (`Analytics`), график продаж / активности
- **Проблема:**
  - При переключении периодов («Last 7 days» / «Last 30 days») график «дёргается» и схлопывается по высоте во время фетчинга новых данных.
  - Повторный рендер графика приводит к сдвигу верстки (**Cumulative Layout Shift**), что ухудшает UX и визуальное восприятие интерфейса.
  - При смене разрядности чисел на оси Y ширина подписи меняется, вызывая горизонтальное дергание графика.

- **Решение:**
  1. Зафиксировать минимальную высоту контейнера графика (`min-h-[300px]` / `min-height`) или задать жесткие пропорции через `aspect-ratio` / `ResponsiveContainer`.
  2. Во время загрузки данных переводить график в состояние фонового обновления (keep previous data / `placeholderData: keepPreviousData` в TanStack Query) либо показывать скелетон (**Skeleton**) фиксированной высоты с `absolute` оверлеем-спиннером.
  3. Для оси Y задать фиксированную ширину (например, `width={50}` в Recharts), чтобы горизонтальные границы графика оставались неподвижными при изменении чисел.
<img width="1145" height="436" alt="image" src="https://github.com/user-attachments/assets/415e7e17-5ffc-4deb-89bf-3236eb723f02" />
