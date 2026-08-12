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
