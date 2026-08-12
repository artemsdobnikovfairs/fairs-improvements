# fairs-improvements

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
