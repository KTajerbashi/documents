# آموزش کامل Route و Router در Angular 22

> راهنمای کامل مسیریابی (Routing) در Angular شامل مفاهیم، پارامترها، ناوبری، Guard، Resolver، Lazy Loading و ساختار پیشنهادی پروژه.

## فهرست

1. [مفاهیم پایه](#۱-مفاهیم-پایه)
2. [راه‌اندازی](#۲-راه‌اندازی)
3. [ویژگی‌های تعریف Route](#۳-ویژگی‌های-تعریف-route)
4. [انواع پارامتر](#۴-انواع-پارامتر)
5. [ناوبری](#۵-ناوبری)
6. [مسیرهای تو در تو و Layout](#۶-مسیرهای-تو-در-تو-child-routes-و-layout)
7. [Lazy Loading](#۷-lazy-loading)
8. [Guards](#۸-guards-نگهبان‌ها)
9. [Resolver](#۹-resolver-دریافت-داده-قبل-از-نمایش)
10. [Router Events](#۱۰-router-events-نمایش-loading-سراسری)
11. [Breadcrumb](#۱۱-breadcrumb-با-data)
12. [نکات مهم رفتاری Router](#۱۲-نکات-مهم-رفتاری-router)
13. [ساختار پیشنهادی پروژه](#۱۳-ساختار-پیشنهادی-feature-based-مناسب-clean--ddd)
14. [مثال کامل](#۱۴-مثال-کامل)
15. [خلاصه انتخاب ابزار](#خلاصه-انتخاب-ابزار)
16. [تغییرات مهم Angular 22](#تغییرات-مهم-angular-22)

---

## ۱. مفاهیم پایه

| مفهوم | توضیح |
|---|---|
| **Route** | یک تعریف که می‌گوید «این آدرس ← این کامپوننت». |
| **Routes** | آرایه‌ای از Routeها (جدول مسیرها). |
| **Router** | سرویسی که ناوبری را انجام می‌دهد و وضعیت آدرس را مدیریت می‌کند. |
| **ActivatedRoute** | اطلاعات روت فعلی (پارامترها، data و...). |
| **RouterOutlet** | جایی در تمپلیت که کامپوننت روت فعال در آن رندر می‌شود. |
| **RouterLink** | دایرکتیو لینک برای ناوبری بدون reload صفحه. |

روند کار:

```
کاربر آدرس را عوض می‌کند → Router آدرس را با Routes تطبیق می‌دهد
→ Guardها اجرا می‌شوند → Resolverها اجرا می‌شوند
→ کامپوننت در RouterOutlet ساخته می‌شود
```

---

## ۲. راه‌اندازی

### app.routes.ts

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'home' },
  { path: 'home', loadComponent: () => import('./home/home').then(m => m.Home) },
  { path: '**', loadComponent: () => import('./not-found/not-found').then(m => m.NotFound) },
];
```

### app.config.ts

```typescript
import { ApplicationConfig } from '@angular/core';
import {
  provideRouter,
  withComponentInputBinding,
  withInMemoryScrolling,
  withPreloading,
  PreloadAllModules,
  withViewTransitions,
} from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),                                     // bind شدن پارامترها به input
      withInMemoryScrolling({ scrollPositionRestoration: 'enabled' }), // برگشت به بالای صفحه
      withPreloading(PreloadAllModules),                               // پیش‌بارگذاری lazy routeها
      withViewTransitions(),                                           // انیمیشن بین صفحات (اختیاری)
    ),
  ],
};
```

### app.ts

```typescript
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a routerLink="/home" routerLinkActive="active">خانه</a>
    </nav>
    <router-outlet />
  `,
})
export class App {}
```

---

## ۳. ویژگی‌های تعریف Route

```typescript
{
  path: 'products',                 // بخش آدرس
  component: ProductList,           // کامپوننت (eager)
  loadComponent: () => import(...), // یا lazy
  title: 'محصولات',                 // عنوان تب مرورگر
  data: { breadcrumb: 'محصولات' },  // داده‌ی ثابت
  canActivate: [authGuard],         // گارد ورود
  canDeactivate: [unsavedGuard],    // گارد خروج
  canMatch: [featureGuard],         // گارد match شدن
  resolve: { products: productsResolver }, // دریافت داده قبل از نمایش
  providers: [ProductStore],        // سرویس مخصوص این روت و فرزندانش
  children: [ ... ],                // مسیرهای تو در تو
}
```

### pathMatch

```typescript
{ path: '', pathMatch: 'full', redirectTo: 'home' }  // فقط وقتی آدرس دقیقاً خالی است
{ path: '', pathMatch: 'prefix', ... }               // پیش‌فرض: با هر آدرسی match می‌شود
```

> برای `redirectTo` روی مسیر خالی همیشه `pathMatch: 'full'` بگذارید، وگرنه حلقه‌ی redirect یا رفتار غیرمنتظره پیش می‌آید.

### Wildcard

```typescript
{ path: '**', component: NotFound }  // همیشه آخرین روت
```

ترتیب Routeها مهم است. Router از بالا به پایین می‌گردد و اولین match را برمی‌دارد.

### title دینامیک

```typescript
{
  path: 'manage/:guid',
  title: (route) => `مدیریت ${route.paramMap.get('guid')}`,
  // یا ResolveFn
}
```

---

## ۴. انواع پارامتر

### الف) Path Parameter (اجباری، بخشی از آدرس)

```typescript
{ path: 'manage/:guid', loadComponent: ... }
// آدرس: /manage/3f2504e0-...
```

### ب) Query Parameter (اختیاری، بعد از `?`)

```
/products?page=2&sort=price
```

### ج) Fragment (بعد از `#`)

```
/docs#section-3
```

### د) Route Data (ثابت، در تعریف روت)

```typescript
data: { roles: ['admin'] }
```

### ه) Navigation State (مخفی، در history)

```typescript
this.router.navigate(['/detail'], { state: { from: 'list' } });
```

> state در history مرورگر می‌ماند ولی در لینک اشتراکی نیست. برای اطلاعات مهم از آن استفاده نکنید.

### خواندن پارامترها

**روش مدرن: Input Binding** (نیاز به `withComponentInputBinding()`):

```typescript
export class ProductDetail {
  id    = input.required<string>();   // path param :id
  page  = input<string>('1');         // query param ?page=
  title = input<string>();            // از data یا resolve هم می‌آید
}
```

**روش کلاسیک: ActivatedRoute**:

```typescript
private route = inject(ActivatedRoute);

// Observable، با تغییر پارامتر دوباره emit می‌شود
this.route.paramMap.subscribe(p => p.get('id'));
this.route.queryParamMap.subscribe(q => q.get('page'));

// Snapshot، فقط یک بار و با reuse کامپوننت به‌روز نمی‌شود
const id = this.route.snapshot.paramMap.get('id');
```

**تبدیل به signal:**

```typescript
guid = toSignal(
  this.route.paramMap.pipe(map(p => p.get('guid'))),
  { initialValue: null },
);
```

---

## ۵. ناوبری

### در تمپلیت (RouterLink)

```html
<!-- ثابت -->
<a routerLink="/home">خانه</a>

<!-- با پارامتر (آرایه‌ای، امن‌تر از string) -->
<a [routerLink]="['/workflow-vehicle-license/manage', item.guid]">مدیریت</a>

<!-- با query و fragment -->
<a [routerLink]="['/products']"
   [queryParams]="{ page: 2, sort: 'price' }"
   fragment="top"
   queryParamsHandling="merge">صفحه ۲</a>

<!-- حالت active -->
<a routerLink="/home"
   routerLinkActive="active"
   [routerLinkActiveOptions]="{ exact: true }">خانه</a>
```

### در کد (Router)

```typescript
private router = inject(Router);
private route = inject(ActivatedRoute);

// آرایه‌ای
this.router.navigate(['/workflow-vehicle-license/manage', guid]);

// آدرس string کامل
this.router.navigateByUrl('/home');

// نسبی نسبت به روت فعلی
this.router.navigate(['../request'], { relativeTo: this.route });

// فقط تغییر query بدون تغییر مسیر
this.router.navigate([], {
  relativeTo: this.route,
  queryParams: { page: 3 },
  queryParamsHandling: 'merge',   // حفظ بقیه‌ی queryها
});

// بدون ثبت در history (مثل بعد از login)
this.router.navigate(['/home'], { replaceUrl: true });

// نتیجه‌ی ناوبری
const ok = await this.router.navigate(['/x']);  // true / false (گارد رد کرده)
```

---

## ۶. مسیرهای تو در تو (Child Routes) و Layout

```typescript
export const routes: Routes = [
  {
    path: '',
    component: MainLayout,          // شامل هدر، منو و <router-outlet />
    children: [
      { path: 'home', loadComponent: ... },
      { path: 'workflow-vehicle-license', loadChildren: () =>
          import('./workflows/workflows.routes').then(m => m.WorkflowsRoutes) },
    ],
  },
  {
    path: 'auth',
    component: AuthLayout,          // Layout دیگر برای صفحات login
    children: [
      { path: 'login', loadComponent: ... },
    ],
  },
];
```

```html
<!-- main-layout.html -->
<app-header />
<aside>...</aside>
<main><router-outlet /></main>
```

هر کامپوننتی که `children` دارد باید `<router-outlet />` داشته باشد.

### مسیر بدون کامپوننت (فقط گروه‌بندی)

```typescript
{
  path: 'workflow-vehicle-license',
  children: [ { path: 'request', ... }, { path: 'manage/:guid', ... } ],
}
```

---

## ۷. Lazy Loading

```typescript
// یک کامپوننت
{ path: 'about', loadComponent: () => import('./about/about').then(m => m.About) }

// یک گروه کامل از روت‌ها
{ path: 'orders', loadChildren: () => import('./orders/orders.routes').then(m => m.OrdersRoutes) }
```

برای `loadChildren` فایل `orders.routes.ts` فقط `export const OrdersRoutes: Routes = [...]` دارد. هر کدام در bundle جداگانه ساخته می‌شود و فقط وقتی کاربر به آن مسیر برود دانلود می‌شود. این ساختار با Microservice-style Frontend (feature-based) هم‌خوان است.

---

## ۸. Guards (نگهبان‌ها)

همه‌ی Guardها در Angular مدرن **تابع** هستند (با `inject` استفاده می‌شود).

| Guard | زمان اجرا | کاربرد |
|---|---|---|
| `canMatch` | قبل از match شدن روت و دانلود chunk | feature flag، نقش کاربر |
| `canActivate` | قبل از ورود به روت | login، بررسی پارامتر |
| `canActivateChild` | قبل از ورود به فرزندان | حفاظت از یک گروه |
| `canDeactivate` | قبل از خروج از روت | تغییرات ذخیره‌نشده |

### مثال: Auth Guard

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.isLoggedIn()
    ? true
    : new RedirectCommand(
        router.createUrlTree(['/auth/login'], { queryParams: { returnUrl: state.url } }),
      );
};
```

### مثال: Role Guard با data

```typescript
export const roleGuard: CanActivateFn = (route) => {
  const roles = route.data['roles'] as string[];
  return inject(AuthService).hasAnyRole(roles);
};

// استفاده
{ path: 'admin', canActivate: [roleGuard], data: { roles: ['admin'] }, ... }
```

### مثال: اعتبارسنجی GUID

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, RedirectCommand, Router } from '@angular/router';

const GUID_REGEX =
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export const validGuidGuard: CanActivateFn = (route) => {
  const router = inject(Router);
  const guid = route.paramMap.get('guid');

  return guid && GUID_REGEX.test(guid)
    ? true
    : new RedirectCommand(router.parseUrl('/workflow-vehicle-license/request'));
};
```

### مثال: canDeactivate

```typescript
export interface HasUnsavedChanges { hasChanges(): boolean; }

export const unsavedGuard: CanDeactivateFn<HasUnsavedChanges> = (component) =>
  component.hasChanges() ? confirm('تغییرات ذخیره نشده، خارج می‌شوید؟') : true;
```

### مقدار بازگشتی Guard

- `true`: اجازه
- `false`: رد (در همان صفحه می‌ماند)
- `RedirectCommand` یا `UrlTree`: هدایت به جای دیگر
- هر کدام به‌صورت `Promise` یا `Observable` هم پذیرفته می‌شود

---

## ۹. Resolver (دریافت داده قبل از نمایش)

```typescript
export const licenseResolver: ResolveFn<LicenseDto> = (route) => {
  const router = inject(Router);
  const guid = route.paramMap.get('guid')!;

  return inject(LicenseApi).getByGuid(guid).pipe(
    catchError(() => of(new RedirectCommand(router.parseUrl('/not-found')))),
  );
};
```

```typescript
{
  path: 'manage/:guid',
  resolve: { license: licenseResolver },
  loadComponent: ...
}
```

```typescript
export class VehicleLicenseManage {
  guid    = input.required<string>();
  license = input.required<LicenseDto>();   // نام کلید resolve
}
```

**مزیت:** کامپوننت با داده‌ی آماده باز می‌شود.
**عیب:** تا پایان دریافت داده، ناوبری معلق می‌ماند (می‌توانید Loading سراسری نشان دهید). برای حالت سریع‌تر از `resource()` داخل کامپوننت استفاده کنید.

---

## ۱۰. Router Events (نمایش Loading سراسری)

```typescript
@Component({ ... })
export class App {
  private router = inject(Router);

  isNavigating = toSignal(
    this.router.events.pipe(
      filter(e =>
        e instanceof NavigationStart ||
        e instanceof NavigationEnd ||
        e instanceof NavigationCancel ||
        e instanceof NavigationError),
      map(e => e instanceof NavigationStart),
    ),
    { initialValue: false },
  );
}
```

```html
@if (isNavigating()) { <app-progress-bar /> }
<router-outlet />
```

---

## ۱۱. Breadcrumb با data

```typescript
data: { breadcrumb: 'manage' }
```

```typescript
// خواندن زنجیره‌ی روت‌ها
buildBreadcrumbs(route: ActivatedRoute): string[] {
  const result: string[] = [];
  let current: ActivatedRoute | null = route.root;
  while (current) {
    const crumb = current.snapshot.data['breadcrumb'];
    if (crumb) result.push(crumb);
    current = current.firstChild;
  }
  return result;
}
```

با رویداد `NavigationEnd` این تابع را فراخوانی کنید.

---

## ۱۲. نکات مهم رفتاری Router

1. **Reuse کامپوننت:** رفتن از `/manage/1` به `/manage/2` کامپوننت را دوباره نمی‌سازد. پس `snapshot` به‌روز نمی‌شود، از signal input یا `paramMap` استفاده کنید.
2. **Angular 22:** `paramsInheritanceStrategy` پیش‌فرض `'always'` شده است. فرزندان پارامترهای والدین را هم می‌بینند. اگر نام پارامتر تکراری است (مثلاً `id` در والد و فرزند) مراقب باشید.
3. **ترتیب Routeها** مهم است. روت‌های خاص‌تر قبل از عمومی‌تر و `**` آخر.
4. **`onSameUrlNavigation`:** پیش‌فرض ignore است. با `withRouterConfig({ onSameUrlNavigation: 'reload' })` می‌توانید تغییرش دهید.

---

## ۱۳. ساختار پیشنهادی (Feature-based، مناسب Clean / DDD)

```
src/app/
├─ app.routes.ts                  ← فقط روت‌های سطح بالا + loadChildren
├─ core/
│  └─ guards/ (auth.guard.ts, valid-guid.guard.ts)
├─ layouts/ (main-layout, auth-layout)
└─ features/
   └─ workflows/
      ├─ workflows.routes.ts      ← WorkflowsRoutes
      └─ workflow-vehicle-license/
         ├─ vehicle-license-request/
         ├─ vehicle-license-manage/
         ├─ data/  (resolvers, api)
         └─ domain/
```

قواعد:

- هر feature فایل `*.routes.ts` خودش را دارد.
- Guard و Resolver پارامتر را اعتبارسنجی می‌کنند و کامپوننت فقط `input` دریافت می‌کند.
- کامپوننت به `ActivatedRoute` وابسته نباشد، تا تست و استفاده‌ی مجدد ساده‌تر شود.
- پارامتر روت در لایه‌ی Presentation فقط یک مقدار ساده (ID) است. آن را اعتبارسنجی کنید و به Facade یا UseCase بدهید.

---

## ۱۴. مثال کامل

### workflows.routes.ts

```typescript
export const WorkflowsRoutes: Routes = [
  {
    path: 'workflow-vehicle-license',
    canActivate: [authGuard],
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'request' },
      {
        path: 'request',
        title: 'درخواست مجوز خودرو',
        loadComponent: () =>
          import('./workflow-vehicle-license/vehicle-license-request/vehicle-license-request')
            .then(m => m.WorkflowVehicleLicense),
        canDeactivate: [unsavedGuard],
        data: { breadcrumb: 'درخواست' },
      },
      { path: 'manage', pathMatch: 'full', redirectTo: 'request' },
      {
        path: 'manage/:guid',
        title: 'مدیریت مجوز',
        canActivate: [validGuidGuard],
        resolve: { license: licenseResolver },
        loadComponent: () =>
          import('./workflow-vehicle-license/vehicle-license-manage/vehicle-license-manage')
            .then(m => m.VehicleLicenseManage),
        data: { breadcrumb: 'مدیریت' },
      },
    ],
  },
];
```

### کامپوننت مدیریت

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'rg-vehicle-license-manage',
  imports: [],
  templateUrl: './vehicle-license-manage.html',
  styleUrl: './vehicle-license-manage.scss',
})
export class VehicleLicenseManage {
  guid    = input.required<string>();     // از :guid
  license = input.required<LicenseDto>(); // از resolve
}
```

```html
<p>vehicle-license-manage works!</p>
<p>GUID: {{ guid() }}</p>
```

### لیست درخواست‌ها: رفتن به صفحه‌ی مدیریت

```typescript
@Component({
  template: `
    @for (item of items(); track item.guid) {
      <a [routerLink]="['/workflow-vehicle-license/manage', item.guid]">{{ item.title }}</a>
    }
  `,
})
export class RequestList { items = signal<LicenseItem[]>([]); }
```

### ترتیب اجرا

هنگام باز کردن `/workflow-vehicle-license/manage/<guid>`:

```
authGuard ← validGuidGuard ← licenseResolver ← ساخت کامپوننت با inputهای guid و license
```

### تست

```
/workflow-vehicle-license/manage/3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

---

## خلاصه انتخاب ابزار

| نیاز | ابزار |
|---|---|
| خواندن پارامتر | `input()` + `withComponentInputBinding()` |
| لینک در تمپلیت | `routerLink` با آرایه |
| ناوبری در کد | `router.navigate()` |
| محافظت از مسیر | Guard تابعی |
| داده‌ی لازم قبل از نمایش | `resolve` |
| بارگذاری تنبل | `loadComponent` / `loadChildren` |
| Loading سراسری | `router.events` |

---

## تغییرات مهم Angular 22

- **`paramsInheritanceStrategy` پیش‌فرض `'always'`:** پارامترهای همه‌ی روت‌های والد به فرزندان می‌رسد. برای بازگشت به رفتار قبلی:

  ```typescript
  provideRouter(routes, withRouterConfig({ paramsInheritanceStrategy: 'emptyOnly' }))
  ```

- **`CanMatch` کلاسی:** پارامتر `currentSnapshot` اجباری شده است (Guardهای تابعی تحت تأثیر نیستند).
- **`isActive()`:** تابعی که یک signal برمی‌گرداند و وضعیت فعال بودن یک لینک را نشان می‌دهد.
- **Route Resources:** با `withRouterResources()` می‌توان داده را با Resource API و به‌صورت reactive دریافت کرد.
- **`withAutoCleanupInjectors()`:** پاک‌سازی خودکار providerهای سطح روت پس از خروج از آن (طبق منابع، در 22.2 پایدار شده است).

> برای جزئیات دقیق به مستندات رسمی Angular در [angular.dev/guide/routing](https://angular.dev/guide/routing) مراجعه کنید.
