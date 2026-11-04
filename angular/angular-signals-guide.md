# راهنمای کامل Signals در آنگولار

> مستندات جامع از مباحث: `input()` در برابر `@Input()`، `get/set`، بررسی کد، `signal` در برابر `model`، و آموزش کامل Signal API همراه با نمونه‌کد، دلیل استفاده و اشتباهات رایج.

## فهرست

1. [چرا Signal؟ (پایه‌ی درک)](#1-چرا-signal)
2. [مزایای `input()` نسبت به `@Input()`](#2-مزایای-input-نسبت-به-input)
3. [مزیت و محدودیت `get/set`](#3-مزیت-و-محدودیت-getset)
4. [بررسی کد: wrapper روی signal](#4-بررسی-کد-wrapper-روی-signal)
5. [آیا برای حذف `()` از template، getter بنویسیم؟](#5-آیا-برای-حذف--از-template-getter-بنویسیم)
6. [مثال اصولی: کامپوننت با route و `guid`](#6-مثال-اصولی-کامپوننت-با-route-و-guid)
7. [`signal()`](#7-signal)
8. [`computed()`](#8-computed)
9. [`effect()`](#9-effect)
10. [`untracked()`](#10-untracked)
11. [`linkedSignal()`](#11-linkedsignal)
12. [`input()`](#12-input)
13. [`output()`](#13-output)
14. [`model()` و مقایسه با `signal`](#14-model-و-مقایسه-با-signal)
15. [Signal Queries](#15-signal-queries)
16. [تعامل با RxJS](#16-تعامل-با-rxjs)
17. [`resource()` و `httpResource()`](#17-resource-و-httpresource)
18. [نقشه‌ی تصمیم‌گیری](#18-نقشه-تصمیم-گیری)
19. [مثال یکپارچه با Clean Architecture / DDD](#19-مثال-یکپارچه-با-clean-architecture--ddd)
20. [اشتباهات رایج](#20-اشتباهات-رایج)
21. [مسیر یادگیری پیشنهادی](#21-مسیر-یادگیری-پیشنهادی)

---

## 1. چرا Signal؟

### مشکل قدیمی: zone.js

آنگولار قدیمی نمی‌دانست **کدام داده تغییر کرده**. برای همین `zone.js` همه‌ی رویدادها (کلیک، `setTimeout`، `Promise`، HTTP) را شنود می‌کرد و بعد از هر کدام **کل درخت کامپوننت‌ها** را بررسی می‌کرد (Change Detection).

```
کلیک → zone.js خبر می‌دهد → آنگولار همه‌ی کامپوننت‌ها را چک می‌کند
```

هزینه‌ها: عملکرد پایین‌تر در برنامه‌های بزرگ، دیباگ سخت، حجم بیشتر bundle.

### راه‌حل: Signal

Signal یک **جعبه‌ی مقدار است که می‌داند چه کسی آن را خوانده**. وقتی مقدارش عوض شود، دقیقاً همان مصرف‌کننده‌ها (template، computed، effect) را خبر می‌کند.

```
signal تغییر کرد → فقط template/computed هایی که آن را خوانده‌اند به‌روز می‌شوند
```

به این مدل می‌گویند **Fine-grained Reactivity**.

### مفاهیم کمکی

| مفهوم | توضیح |
|---|---|
| **Change Detection** | فرایند مقایسه‌ی state با DOM و به‌روزرسانی آن |
| **OnPush** | کامپوننت فقط وقتی چک شود که input عوض شده، event داخلی داشته، یا signalاش تغییر کرده باشد |
| **Zoneless** | حذف کامل zone.js و تکیه بر signal (در نسخه‌های جدید پشتیبانی می‌شود) |
| **Reactive Graph** | گراف وابستگی: signal ← computed ← template |
| **Push/Pull** | signal هم تغییر را اطلاع می‌دهد (push) و هم مقدار را وقتی لازم است محاسبه می‌کند (pull / lazy) |

> برای پروژه‌های جدید: `ChangeDetectionStrategy.OnPush` را برای همه‌ی کامپوننت‌ها بگذارید. با signal کاملاً سازگار است و مسیر ورود به zoneless را باز می‌کند.

---

## 2. مزایای `input()` نسبت به `@Input()`

### ۲.۱ واکنش‌پذیری واقعی

```typescript
// روش قدیمی
@Input() price = 0;
@Input() qty = 1;
total = 0;

ngOnChanges() {
  this.total = this.price * this.qty;
}

// روش جدید
price = input(0);
qty = input(1);
total = computed(() => this.price() * this.qty());
```

`total` خودکار و فقط وقتی لازم است دوباره محاسبه می‌شود (lazy و memoized).

### ۲.۲ سازگاری با Zoneless و OnPush

Change Detection دقیقاً می‌داند کدام کامپوننت به کدام مقدار وابسته است؛ عملکرد بهتر و مسیر آماده برای حذف `zone.js`.

### ۲.۳ Type Safety بهتر

```typescript
name = input.required<string>();                          // خطا در زمان کامپایل اگر داده نشود
disabled = input(false, { transform: booleanAttribute }); // تبدیل مقدار
userId = input<number>(0, { alias: 'id' });               // نام مستعار
```

در `input.required` مقدار هرگز `undefined` نیست.

### ۲.۴ حذف نیاز به `ngOnChanges`

منطق وابسته به input را به‌جای پخش‌کردن در `ngOnInit`، `ngOnChanges` و setterها، declarative با `computed` و `effect` می‌نویسید.

### ۲.۵ Read-only بودن

`this.name.set(...)` برای input وجود ندارد؛ جریان داده یک‌طرفه و قابل پیش‌بینی می‌ماند.

### مقایسه

| مورد | `@Input()` ساده | `input()` |
|---|---|---|
| تشخیص تغییر | وابسته به Zone/ngOnChanges | دقیق و Signal-based |
| مقدار مشتق‌شده | دستی | `computed` خودکار |
| Required | ضعیف | `input.required` |
| Zoneless | مشکل‌دار | کاملاً سازگار |
| Immutable | خیر | بله |

> `@Input()` منسوخ نشده و کار می‌کند، ولی مسیر توصیه‌شده‌ی آنگولار signal-based است (از Angular 17.1).

---

## 3. مزیت و محدودیت `get/set`

Getter و setter اجازه می‌دهند هنگام خواندن/نوشتن property منطق اجرا شود، بدون تغییر شکل استفاده‌ی بیرونی.

### کاربردها

**اعتبارسنجی و پاک‌سازی ورودی**

```typescript
private _name = '';

@Input()
set name(value: string) {
  this._name = value?.trim().toUpperCase() ?? '';
}
get name(): string {
  return this._name;
}
```

**واکنش به تغییر Input بدون `ngOnChanges`**

```typescript
@Input()
set userId(id: number) {
  this._userId = id;
  this.loadUser(id);
}
```

**Property محاسبه‌شده**

```typescript
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

**کپسوله‌سازی:** فیلد private و دسترسی کنترل‌شده، یا property فقط‌خواندنی با ارائه‌ی فقط `get`.

### ⚠️ احتیاط‌ها

- Getter در template در **هر چرخه‌ی Change Detection** اجرا می‌شود. محاسبه‌ی سنگین باعث افت عملکرد می‌شود؛ به‌جای آن `computed` یا `pure pipe`.
- در setter کار سنگین و side effect زیاد نکنید.
- Getter و setter بهتر است type یکسان داشته باشند.

### مقایسه با Signal

| نیاز | بهترین انتخاب |
|---|---|
| مقدار مشتق‌شده | `computed()` |
| واکنش به تغییر Input | `effect()` یا `computed()` |
| تبدیل مقدار ورودی | `input(..., { transform })` |
| پروژه‌ی قدیمی (قبل از v17.1) | `get/set` |

---

## 4. بررسی کد: wrapper روی signal

### کد بررسی‌شده

```typescript
name = signal<string>('');

get getName(): string {
  return this.name();
}

set setName(value: string) {
  this.name.set(value);
}
```

### نتیجه: کار می‌کند، ولی اصولی نیست

1. **Redundant Abstraction:** signal خودش API کامل دارد (`name()` و `name.set()`)؛ wrapper ارزشی اضافه نمی‌کند.
2. **نام‌گذاری غلط:** در TypeScript get/set یک property با **یک نام** تعریف می‌شوند. `getName` و `setName` دو property مجزا هستند و استفاده‌شان عجیب است (`this.setName = 'Ali'`)؛ سبک Java است.
3. **پنهان شدن signal بودن:** بیرون نمی‌داند با signal طرف است و نمی‌تواند مستقیم در `computed` استفاده کند.
4. **Encapsulation ظاهری:** `name` همچنان public است و `name.set()` مستقیم قابل‌فراخوانی است.

### روش‌های اصولی

**حالت ۱: ساده (بیشتر موارد)**

```typescript
name = signal('');
// خواندن: this.name()
// نوشتن:  this.name.set('Ali')
```

**حالت ۲: فقط‌خواندنی برای بیرون + متد دارای منطق**

```typescript
private readonly _name = signal('');
readonly name = this._name.asReadonly();

setName(value: string) {
  this._name.set(value.trim());
}
```

**حالت ۳: مقدار مشتق‌شده**

```typescript
firstName = signal('');
lastName = signal('');
fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
```

**حالت ۴: سازگاری با `@Input` قدیمی (هر دو با یک نام)**

```typescript
private readonly _name = signal('');

@Input()
set name(value: string) { this._name.set(value); }
get name(): string { return this._name(); }
```

**حالت ۵: نسخه‌های جدید**

```typescript
name = input('');   // ورودی فقط‌خواندنی
name = model('');   // two-way binding
```

> **قاعده‌ی کلی:** wrapper فقط وقتی بنویسید که **منطق اضافه** (validation، transform، قانون دامنه) داشته باشد.

---

## 5. آیا برای حذف `()` از template، getter بنویسیم؟

سوال: به‌جای `{{ name() }}` از `{{ getName }}` استفاده کنیم؟

**پاسخ: توجیه خوبی ندارد.**

- فقط تغییر ظاهری است و برای هر signal یک getter Boilerplate اضافه می‌کند.
- با قرارداد جامعه‌ی Angular مغایرت دارد؛ `()` به خواننده می‌گوید «این signal است».
- دو راه برای یک چیز، ذهن را درگیر می‌کند (در `computed` و child باید `name()` نوشت).
- getter در template معمولاً علامت هشدار است و خواننده را نگران می‌کند.
- `()` عمداً بخشی از مدل ذهنی signal است (`input()`, `computed()`, `model()`, `viewChild()`, `resource()`).

### جایگزین‌های تمیز

**`@let` (Angular 18.1+)** برای خواندن چندباره:

```html
@let userName = name();
<h1>{{ userName }}</h1>
<p>Hello {{ userName }}</p>
```

**`@if` با alias:**

```html
@if (user(); as u) {
  <div>{{ u.name }}</div>
}
```

---

## 6. مثال اصولی: کامپوننت با route و `guid`

### مشکلات کد اولیه

1. `ngOnInit` خالی؛ باید حذف شود.
2. پیشوند `_` برای accessor عمومی قرارداد را می‌شکند (`_` برای فیلد private است).
3. get/set هیچ منطقی اضافه نمی‌کند.
4. `guid` درست است ولی به تنظیم router نیاز دارد.

### کامپوننت اصلاح‌شده

```typescript
import { ChangeDetectionStrategy, Component, computed, inject, input, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';
import { VehicleLicenseApi } from './vehicle-license.api';

@Component({
  selector: 'rg-vehicle-license-manage',
  templateUrl: './vehicle-license-manage.html',
  styleUrl: './vehicle-license-manage.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class VehicleLicenseManage {
  private readonly api = inject(VehicleLicenseApi);

  // ۱) ورودی از route (فقط‌خواندنی، الزامی)
  readonly guid = input.required<string>();

  // ۲) داده‌ی سرور: با هر تغییر guid خودکار دوباره لود می‌شود
  readonly license = toSignal(
    toObservable(this.guid).pipe(switchMap(guid => this.api.getByGuid(guid)))
  );

  // ۳) state فرم: قابل‌نوشتن فقط داخل کلاس
  private readonly _licenseNumber = signal('');
  readonly licenseNumber = this._licenseNumber.asReadonly();

  // ۴) مقدار مشتق‌شده
  readonly isValid = computed(() => this._licenseNumber().length >= 5);

  // ۵) تغییر state فقط از مسیر کنترل‌شده
  setLicenseNumber(value: string): void {
    this._licenseNumber.set(value.trim().toUpperCase());
  }
}
```

### Template

```html
<h2>مدیریت گواهی: {{ guid() }}</h2>

@if (license(); as l) {
  <p>مالک: {{ l.ownerName }}</p>
}

<input
  [value]="licenseNumber()"
  (input)="setLicenseNumber($any($event.target).value)" />

@if (!isValid()) {
  <small>شماره باید حداقل ۵ کاراکتر باشد</small>
}
```

### تنظیم لازم Router

```typescript
// app.config.ts
provideRouter(routes, withComponentInputBinding())

// routes
{ path: 'vehicle-license/:guid', component: VehicleLicenseManage }
```

نام input باید دقیقاً با `:guid` یکی باشد؛ در غیر این‌صورت از `alias` استفاده کنید: `input.required<string>({ alias: 'guid' })`.

### چرا این‌طور نوشتیم؟

| تصمیم | دلیل |
|---|---|
| `input.required` | اگر route درست تنظیم نشود، خطا همان لحظه مشخص می‌شود |
| `toObservable` + `switchMap` | با تغییر guid، درخواست قبلی لغو و داده‌ی جدید لود می‌شود (کامپوننت reuse می‌شود) |
| `private` + `asReadonly()` | Encapsulation واقعی |
| متد `setLicenseNumber` | چون `trim` و `toUpperCase` دارد، ارزش واقعی اضافه می‌کند |
| `computed` برای `isValid` | وابستگی را ردیابی می‌کند و فقط در صورت لزوم محاسبه می‌شود |
| `OnPush` | سازگار با signal و رندر حداقلی |
| حذف `ngOnInit` | همه‌چیز declarative است |

### کی کدام؟

```typescript
count = signal(0);                                   // state ساده بدون قانون
private readonly _x = signal('');                    // state با قانون
readonly x = this._x.asReadonly();
total = computed(() => this.price() * this.qty());   // مشتق‌شده
value = model('');                                   // two-way
```

### get/set کجا هنوز درست است؟

1. کد قدیمی با `@Input() set x(...)` که هنوز مهاجرت نکرده.
2. کلاس‌های غیر component (مثل Entity یا Value Object در لایه‌ی Domain).

---

## 7. `signal()`

**مفهوم:** مخزن مقدار واکنش‌گرا و قابل‌نوشتن.
**چرا:** تغییرش خودکار به template و وابسته‌ها برسد، بدون `ngOnChanges`، `markForCheck` یا `subscribe`.

```typescript
count = signal(0);

count();                    // خواندن
count.set(5);               // مقداردهی
count.update(c => c + 1);   // بر اساس مقدار قبلی
```

### Immutability (بسیار مهم)

Signal تغییر را با مقایسه‌ی reference (`Object.is`) تشخیص می‌دهد.

```typescript
user = signal({ name: 'Ali', age: 30 });

// ❌ غلط: reference یکی است
this.user().age = 31;

// ✅ درست
this.user.update(u => ({ ...u, age: 31 }));

// آرایه
items = signal<string[]>([]);
this.items.update(list => [...list, 'new']);              // افزودن
this.items.update(list => list.filter(x => x !== 'a'));   // حذف
```

### Equality سفارشی

```typescript
point = signal({ x: 1, y: 2 }, {
  equal: (a, b) => a.x === b.x && a.y === b.y,
});
```

### مثال: Loading State

```typescript
@Component({
  template: `
    <button (click)="save()" [disabled]="isSaving()">ذخیره</button>
    @if (isSaving()) { <span>در حال ذخیره...</span> }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SaveButton {
  protected readonly isSaving = signal(false);

  async save() {
    this.isSaving.set(true);
    try { await this.api.save(); }
    finally { this.isSaving.set(false); }
  }
}
```

---

## 8. `computed()`

**مفهوم:** signal **فقط‌خواندنی** که از signalهای دیگر محاسبه می‌شود.

**چرا:**
- **Lazy:** تا کسی نخواند محاسبه نمی‌شود.
- **Memoized:** تا وابستگی‌ها عوض نشوند، نتیجه cache است.
- **ردیابی خودکار وابستگی:** نیازی به لیست dependency نیست (برخلاف `useMemo` در React).

```typescript
price = signal(100);
qty = signal(2);
discount = signal(0.1);

subtotal = computed(() => this.price() * this.qty());
total = computed(() => this.subtotal() * (1 - this.discount()));
```

### مثال: فیلتر لیست

```typescript
products = signal<Product[]>([]);
search = signal('');
onlyInStock = signal(false);

filtered = computed(() => {
  const q = this.search().toLowerCase();
  return this.products().filter(p =>
    p.name.toLowerCase().includes(q) &&
    (!this.onlyInStock() || p.stock > 0)
  );
});

count = computed(() => this.filtered().length);
```

### قوانین

- داخل `computed` **side effect ننویسید** (HTTP، `set` روی signal دیگر).
- باید **pure** باشد.
- وابستگی شرطی مشکلی ندارد؛ فقط signalهای خوانده‌شده در اجرای فعلی ردیابی می‌شوند.

**چرا بهتر از getter؟** getter در هر Change Detection اجرا می‌شود؛ `computed` فقط وقتی وابستگی عوض شود.

---

## 9. `effect()`

**مفهوم:** تابعی که هر بار signalهای خوانده‌شده در آن تغییر کنند، دوباره اجرا می‌شود.
**چرا:** برای کار با **دنیای بیرون از آنگولار**: logging، localStorage، آنالیتیکس، کتابخانه‌ی ثالث، canvas.

```typescript
constructor() {
  effect(() => {
    localStorage.setItem('theme', this.theme());
  });

  effect((onCleanup) => {
    const id = setInterval(() => console.log(this.count()), 1000);
    onCleanup(() => clearInterval(id));   // پاک‌سازی قبل از اجرای بعدی
  });
}
```

### ⚠️ مهم‌ترین قانون

**برای ساختن state مشتق‌شده از effect استفاده نکنید.**

```typescript
// ❌ بد
effect(() => { this.total.set(this.price() * this.qty()); });

// ✅ خوب
total = computed(() => this.price() * this.qty());
```

| نیاز | ابزار |
|---|---|
| مقدار مشتق‌شده | `computed` |
| مشتق‌شده که کاربر هم می‌تواند بنویسد | `linkedSignal` |
| کار روی دنیای بیرونی | `effect` |
| واکنش به رویداد کاربر | متد event handler |
| بارگذاری async | `resource` / RxJS |

effect باید در **injection context** ساخته شود (constructor یا field initializer).

---

## 10. `untracked()`

**مفهوم:** مقدار را بخوان ولی آن را وابستگی حساب نکن.

```typescript
effect(() => {
  const user = this.user();                       // ردیابی می‌شود
  const theme = untracked(() => this.theme());    // ردیابی نمی‌شود
  this.logger.log(user.id, theme);
});
```

فقط تغییر `user` باعث اجرای دوباره می‌شود.

---

## 11. `linkedSignal()`

**مفهوم:** مثل `computed` از signal دیگر مقدار می‌گیرد ولی قابل `set` دستی هم هست؛ با تغییر منبع دوباره reset می‌شود.
**چرا:** مسئله‌ی کلاسیک «لیست گزینه‌ها عوض شد، انتخاب باید به اولین مورد برگردد، ولی کاربر هم می‌تواند انتخاب کند.»

```typescript
brands = signal(['Toyota', 'BMW', 'Kia']);
selected = linkedSignal(() => this.brands()[0]);

this.selected.set('Kia');   // انتخاب کاربر
this.brands.set(['Audi']);  // منبع عوض شد → selected = 'Audi'
```

حفظ انتخاب قبلی اگر هنوز وجود دارد:

```typescript
selected = linkedSignal<string[], string>({
  source: this.brands,
  computation: (list, previous) =>
    previous && list.includes(previous.value) ? previous.value : list[0],
});
```

---

## 12. `input()`

**مفهوم:** signal فقط‌خواندنی که parent مقدارش را می‌دهد.
**چرا:** reactive، type-safe، و قابل استفاده‌ی مستقیم در `computed`.

```typescript
name = input('');                                          // اختیاری
id = input.required<number>();                             // الزامی
active = input(false, { transform: booleanAttribute });    // تبدیل
label = input('', { alias: 'title' });                     // alias
```

```html
<app-card [id]="5" title="سلام" active />
```

input همیشه reactive است:

```typescript
fullName = computed(() => `${this.first()} ${this.last()}`);
```

با `withComponentInputBinding()` پارامتر route، query string و data مستقیم input می‌شوند.

---

## 13. `output()`

**مفهوم:** child به parent رویداد می‌فرستد.
**چرا:** جریان داده یک‌طرفه می‌ماند: **داده به پایین (input)، رویداد به بالا (output)**.

```typescript
saved = output<Order>();
cancelled = output<void>();

save() { this.saved.emit(this.order()); }
```

```html
<app-order-form (saved)="onSaved($event)" (cancelled)="close()" />
```

> `output()` signal نیست؛ emitter ساده است و در پروژه‌های جدید جایگزین `EventEmitter` می‌شود.

---

## 14. `model()` و مقایسه با `signal`

### خلاصه

- **`signal()`**: state **داخلی**؛ فقط خود کامپوننت مالک است.
- **`model()`**: state **مشترک** بین parent و child؛ signal قابل‌نوشتن که هم input است و هم output (`<name>Change`) و two-way binding می‌دهد.

### `model()` در عمل

```typescript
value = model('');                  // input: value  +  output: valueChange
value = model.required<string>();   // الزامی
```

معادل دستی:

```typescript
value = input(0);
valueChange = output<number>();
```

### مثال Child و Parent

```typescript
@Component({
  selector: 'app-counter',
  template: `
    <button (click)="dec()">-</button>
    <span>{{ value() }}</span>
    <button (click)="inc()">+</button>
  `,
})
export class Counter {
  value = model(0);
  inc() { this.value.update(v => v + 1); }
  dec() { this.value.update(v => v - 1); }
}

@Component({
  imports: [Counter],
  template: `
    <app-counter [(value)]="qty" />
    <p>Parent sees: {{ qty() }}</p>
  `,
})
export class Parent {
  qty = signal(1);
}
```

استفاده‌ی یک‌طرفه:

```html
<app-counter [value]="5" />
<app-counter [value]="qty()" (valueChange)="onChange($event)" />
```

### مقایسه‌ی ابزارها

| | `signal()` | `input()` | `model()` | `output()` |
|---|---|---|---|---|
| جهت داده | داخلی | parent → child | parent ⇄ child | child → parent |
| child می‌تواند بنویسد؟ | بله | خیر | بله | emit |
| parent می‌تواند مقدار بدهد؟ | خیر | بله | بله | خیر |
| `.set()` / `.update()` | دارد | ندارد | دارد | ندارد |
| two-way `[( )]` | خیر | خیر | **بله** | خیر |

### کی کدام؟

| سناریو | انتخاب |
|---|---|
| `isLoading`, `isOpen`, تب انتخاب‌شده | `signal()` |
| parent می‌دهد، child فقط نمایش می‌دهد | `input()` |
| رویداد به parent | `output()` |
| کنترل سفارشی (checkbox، rating، datepicker) | `model()` |
| مقدار مشتق‌شده | `computed()` |

قاعده‌ی تصمیم: آیا parent باید مقدار بدهد؟ نه → `signal()`. بله و child تغییر نمی‌دهد → `input()`؛ بله و child تغییر می‌دهد → `model()`.

### نکات مهم

1. `model()` برای state داخلی استفاده نکنید؛ API عمومی می‌سازد.
2. در two-way binding با signal، در template **پرانتز نگذارید**: `[(value)]="qty"` نه `qty()`.
3. مقدار مشتق‌شده را با `computed` بسازید.
4. `model()` گزینه‌ی `transform` ندارد؛ برای قانون، متد بنویسید.
5. مراقب loop در `effect` باشید (نوشتن روی model که همان effect می‌خواند).

### مثال: Rating

```typescript
@Component({
  selector: 'app-rating',
  template: `
    @for (star of stars; track star) {
      <button (click)="rate(star)" [class.active]="star <= value()">★</button>
    }
  `,
})
export class Rating {
  protected readonly stars = [1, 2, 3, 4, 5];
  readonly value = model(0);
  readonly disabled = input(false);

  rate(n: number) {
    if (this.disabled()) return;
    this.value.set(n);
  }
}
```

```html
<app-rating [(value)]="productRating" [disabled]="isReadonly()" />
```

### جمع‌بندی

- **signal** = مال خودم است.
- **input** = از بیرون می‌گیرم، فقط می‌خوانم.
- **output** = رویداد به بیرون می‌فرستم.
- **model** = می‌گیرم، تغییر می‌دهم، و بیرون باخبر می‌شود.

`model()` را برای کامپوننت‌های UI قابل‌استفاده‌ی مجدد (shared/ui) نگه دارید؛ در Smart/Container بیشتر `signal`، `computed` و سرویس‌های state.

---

## 15. Signal Queries

جایگزین `@ViewChild` و `@ContentChild` به‌صورت signal:

```typescript
nameInput = viewChild<ElementRef<HTMLInputElement>>('nameInput');
tabs = viewChildren(TabComponent);
header = contentChild(HeaderDirective);

focus() {
  this.nameInput()?.nativeElement.focus();
}

tabCount = computed(() => this.tabs().length);
```

**چرا بهتر است:** نیازی به `ngAfterViewInit` نیست؛ مقدار reactive است و در `computed` قابل استفاده است.

---

## 16. تعامل با RxJS

Signal جایگزین RxJS نیست.

| RxJS | Signal |
|---|---|
| جریان رویداد در زمان (debounce، switchMap، retry، websocket) | state فعلی و نمایش در UI |
| async و cancellation | مقدار همزمان |

### `toSignal()`

```typescript
users = toSignal(this.http.get<User[]>('/api/users'), { initialValue: [] });
```

اشتراک را خودکار مدیریت و unsubscribe می‌کند.

### `toObservable()`

```typescript
results = toSignal(
  toObservable(this.search).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(q => this.api.search(q)),
  ),
  { initialValue: [] },
);
```

**الگوی طلایی:** signal برای state، RxJS برای عملیات زمانی، و `toSignal` در انتها.

---

## 17. `resource()` و `httpResource()`

**مفهوم:** مدیریت بارگذاری async به‌صورت signal با وضعیت loading و error.

> این API نسبتاً جدید است و بسته به نسخه ممکن است experimental باشد، و نام گزینه‌ها (`params` یا `request`) بین نسخه‌ها فرق کرده است. مستندات نسخه‌ی پروژه را بررسی کنید.

```typescript
guid = input.required<string>();

license = resource({
  params: () => ({ guid: this.guid() }),
  loader: ({ params }) => this.api.getByGuid(params.guid),
});
```

```html
@if (license.isLoading()) { <spinner /> }
@else if (license.error()) { <p>خطا</p> }
@else if (license.value(); as l) { <p>{{ l.ownerName }}</p> }
```

با تغییر `guid` خودکار دوباره لود می‌کند و درخواست قبلی را لغو می‌کند.

---

## 18. نقشه‌ی تصمیم‌گیری

```
state داخلی ساده؟                       → signal()
مشتق از چیز دیگر؟                       → computed()
مشتق ولی کاربر هم باید بتواند عوضش کند؟   → linkedSignal()
از parent می‌گیرم، فقط می‌خوانم؟          → input()
به parent رویداد می‌فرستم؟              → output()
هم می‌گیرم هم تغییر می‌دهم؟             → model()
کار بیرونی (storage/log/canvas)؟        → effect()
عنصر DOM یا child؟                      → viewChild()
async + loading/error؟                  → resource() / RxJS + toSignal
```

---

## 19. مثال یکپارچه با Clean Architecture / DDD

### Store (لایه‌ی Application)

```typescript
// vehicle-license.store.ts
@Injectable()
export class VehicleLicenseStore {
  private readonly api = inject(VehicleLicenseApi);

  // state خصوصی قابل‌نوشتن
  private readonly _items = signal<VehicleLicense[]>([]);
  private readonly _loading = signal(false);
  private readonly _error = signal<string | null>(null);
  readonly search = signal('');

  // نمای فقط‌خواندنی برای بیرون
  readonly items = this._items.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();

  // مقدار مشتق‌شده
  readonly filtered = computed(() => {
    const q = this.search().trim().toLowerCase();
    return this._items().filter(x => x.plateNumber.toLowerCase().includes(q));
  });
  readonly count = computed(() => this.filtered().length);

  // عملیات (use-case)
  async load() {
    this._loading.set(true);
    this._error.set(null);
    try {
      this._items.set(await firstValueFrom(this.api.getAll()));
    } catch {
      this._error.set('بارگذاری ناموفق بود');
    } finally {
      this._loading.set(false);
    }
  }

  remove(id: string) {
    this._items.update(list => list.filter(x => x.id !== id));
  }
}
```

### کامپوننت Smart (Container)

```typescript
@Component({
  selector: 'rg-vehicle-license-list',
  imports: [LicenseTable, SearchBox],
  providers: [VehicleLicenseStore],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <rg-search-box [(value)]="store.search" />

    @if (store.loading()) { <p>در حال بارگذاری...</p> }
    @else if (store.error(); as err) { <p>{{ err }}</p> }
    @else {
      <p>تعداد: {{ store.count() }}</p>
      <rg-license-table
        [items]="store.filtered()"
        (deleted)="store.remove($event)" />
    }
  `,
})
export class VehicleLicenseList implements OnInit {
  protected readonly store = inject(VehicleLicenseStore);
  ngOnInit() { this.store.load(); }
}
```

### کامپوننت‌های Dumb (UI قابل‌استفاده‌ی مجدد)

```typescript
@Component({
  selector: 'rg-search-box',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input [value]="value()" (input)="value.set($any($event.target).value)" />
  `,
})
export class SearchBox {
  value = model('');
}

@Component({
  selector: 'rg-license-table',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @for (i of items(); track i.id) {
      <div>{{ i.plateNumber }}
        <button (click)="deleted.emit(i.id)">حذف</button>
      </div>
    }
  `,
})
export class LicenseTable {
  items = input.required<VehicleLicense[]>();
  deleted = output<string>();
}
```

### نگاشت به Clean/DDD

| بخش | معادل معماری |
|---|---|
| Store | لایه‌ی Application (use-caseها و state) |
| Api | لایه‌ی Infrastructure |
| Container | Smart Component (هماهنگ‌کننده) |
| Dumb components | UI خالص بدون وابستگی به سرویس |
| `asReadonly()` | مرز encapsulation بین لایه‌ها |

---

## 20. اشتباهات رایج

1. **تغییر object به‌صورت mutable** (`this.user().age = 5`). همیشه object جدید بسازید.
2. **استفاده از `effect` برای مقدار مشتق‌شده.** از `computed` استفاده کنید.
3. **فراموشی `()`** هنگام خواندن (`count` به‌جای `count()`).
4. **`model()` برای state داخلی.** `signal()` کافی است.
5. **ساختن effect بیرون از injection context.**
6. **نوشتن wrapper بی‌منطق** (مثل get/set خام روی signal).
7. **فراموشی RxJS** برای debounce، cancel و stream.
8. **عدم استفاده از `track`** در `@for` (عملکرد و حفظ state DOM را خراب می‌کند).
9. **محاسبه‌ی سنگین در effect/computed** بدون توجه به وابستگی‌ها.

---

## 21. مسیر یادگیری پیشنهادی

1. `signal` / `computed` / `effect` و immutability
2. `input` / `output` / `model` و الگوی Smart/Dumb
3. `OnPush` و درک Change Detection
4. `toSignal` / `toObservable` و ترکیب با RxJS
5. `linkedSignal` / `resource`
6. Signal Store (مثل NgRx SignalStore) برای state بزرگ
7. Zoneless (`provideZonelessChangeDetection`)
8. تست کامپوننت‌های signal-based

> **پیشنهاد برای GitHub portfolio:** یک پروژه‌ی کوچک (مثلاً مدیریت گواهی خودرو) با Store مبتنی بر signal، کامپوننت‌های OnPush، یک `model` در shared/ui و یک `resource` برای لود داده بسازید و در README دلیل هر انتخاب را بنویسید.

---

## خلاصه‌ی قواعد طلایی

- wrapper (get/set یا متد) فقط وقتی که **منطق واقعی** دارد.
- state قابل‌نوشتن را **private** نگه دارید و با `asReadonly()` بیرون بدهید.
- مقدار مشتق‌شده = `computed`، نه `effect` و نه getter.
- داده به پایین با `input`، رویداد به بالا با `output`، مقدار مشترک با `model`.
- Signal برای state، RxJS برای stream و زمان.
- همیشه immutable به‌روزرسانی کنید و `OnPush` بگذارید.
