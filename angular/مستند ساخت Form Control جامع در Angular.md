# مستند ساخت Form Control جامع در Angular

راهنمای کاربردی گام‌به‌گام با `ControlValueAccessor`، `BaseControl` و Reactive Forms

## ۱. مقدمه و اهداف

در پروژه‌های واقعی، استفاده‌ی مستقیم از `<input>` در هر فرم باعث تکرار کد (label، پیام خطا، حالت disabled، ARIA و...) می‌شود. راه‌حل این است که یک بار زیرساخت را بنویسیم و بعد `app-input`، `app-select`، `app-textarea` و کنترل‌های دیگر را با حداقل کد بسازیم.

**قابلیت‌های هدف:**

- سازگاری با Reactive Forms و Template-driven (از طریق CVA)
- نمایش خودکار خطا با پیام‌های متمرکز و قابل i18n
- label، hint، placeholder، disabled، readonly
- سازگاری با `markAllAsTouched()` و `reset()` در حالت OnPush
- دسترسی‌پذیری (ARIA)
- نمایش حالت `pending` برای async validator
- فرمت/mask ورودی (مثلاً جداکننده‌ی هزارگان)
- قابل ارث‌بری برای کنترل‌های دیگر

## ۲. مفهوم پایه: ControlValueAccessor

`ControlValueAccessor` (به اختصار CVA) پلی است بین سیستم Forms آنگولار و یک کامپوننت سفارشی. با پیاده‌سازی آن، کامپوننت شما هم با `formControlName` / `formControl` و هم با `ngModel` کار می‌کند.

چهار متد اصلی:

| متد | وظیفه |
| --- | --- |
| `writeValue(v)` | Angular مقدار جدید را به کامپوننت می‌دهد |
| `registerOnChange(fn)` | تابعی که باید با تغییر مقدار صدا بزنیم |
| `registerOnTouched(fn)` | تابعی که هنگام blur صدا می‌زنیم |
| `setDisabledState(b)` | فعال/غیرفعال شدن از بیرون |

## ۳. نمونه‌ی ساده (نقطه‌ی شروع)

```typescript
// app-input.component.ts
import { Component, Input, Optional, Self, signal } from '@angular/core';
import { ControlValueAccessor, NgControl } from '@angular/forms';

@Component({
  selector: 'app-input',
  standalone: true,
  templateUrl: './app-input.component.html',
})
export class AppInputComponent implements ControlValueAccessor {
  @Input() label = '';
  @Input() type: 'text' | 'email' | 'password' | 'number' = 'text';
  @Input() placeholder = '';

  value = signal<string>('');
  disabled = signal(false);

  private onChange: (v: string) => void = () => {};
  private onTouched: () => void = () => {};

  constructor(@Optional() @Self() public ngControl: NgControl) {
    if (this.ngControl) this.ngControl.valueAccessor = this;
  }

  writeValue(value: string): void { this.value.set(value ?? ''); }
  registerOnChange(fn: (v: string) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(d: boolean): void { this.disabled.set(d); }

  onInput(e: Event) {
    const val = (e.target as HTMLInputElement).value;
    this.value.set(val);
    this.onChange(val);
  }
  onBlur() { this.onTouched(); }

  get control() { return this.ngControl?.control; }
  get showError() {
    return !!this.control && this.control.invalid && (this.control.touched || this.control.dirty);
  }
}
```

**دو روش ثبت CVA:**

- **inject کردن `NgControl` با `@Self`** (روش بالا): به `control` و `errors` دسترسی می‌دهد و circular dependency ندارد. پیشنهاد اصلی.
- **provide کردن `NG_VALUE_ACCESSOR`** با `forwardRef` و `multi: true`: ساده‌تر است ولی برای خواندن خطاها باید `Injector` جداگانه بگیرید.

> هر دو را همزمان استفاده نکنید؛ در غیر این صورت خطای circular dependency می‌گیرید.

این نسخه کار می‌کند ولی برای هر کنترل جدید باید همه‌چیز تکرار شود. پس به سراغ معماری قابل استفاده‌ی مجدد می‌رویم.

## ۴. ساختار فایل‌ها

```
shared/forms/
├─ error-messages.token.ts
├─ base-control.ts
├─ field-shell/
│   └─ field-shell.component.ts
└─ app-input/
    ├─ app-input.component.ts
    └─ app-input.component.html
```

## ۵. گام ۱: پیام‌های خطای متمرکز

پیام‌ها را داخل کامپوننت‌ها نگذارید. با یک `InjectionToken` می‌توان آن‌ها را در سطح اپ یا هر ماژول override کرد (برای i18n).

```typescript
// error-messages.token.ts
import { InjectionToken } from '@angular/core';

export type ErrorMessageFn = (error: any, label: string) => string;
export type ErrorMessages = Record<string, ErrorMessageFn>;

export const DEFAULT_ERROR_MESSAGES: ErrorMessages = {
  required: (_, label) => `${label || 'این فیلد'} الزامی است`,
  email: () => 'ایمیل معتبر نیست',
  minlength: (e) => `حداقل ${e.requiredLength} کاراکتر وارد کنید`,
  maxlength: (e) => `حداکثر ${e.requiredLength} کاراکتر مجاز است`,
  min: (e) => `مقدار نباید کمتر از ${e.min} باشد`,
  max: (e) => `مقدار نباید بیشتر از ${e.max} باشد`,
  pattern: () => 'فرمت وارد شده صحیح نیست',
};

export const ERROR_MESSAGES = new InjectionToken<ErrorMessages>('ERROR_MESSAGES', {
  providedIn: 'root',
  factory: () => DEFAULT_ERROR_MESSAGES,
});
```

برای ولیدیتور سفارشی (مثلاً `nationalCode`) فقط یک کلید به این map اضافه کنید.

## ۶. گام ۲: کلاس پایه BaseControl

هسته‌ی اصلی سیستم. هر کنترل جدید فقط از این کلاس ارث می‌برد.

```typescript
// base-control.ts
import {
  ChangeDetectorRef, DestroyRef, Directive, OnInit,
  booleanAttribute, inject, input, signal,
} from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { ControlValueAccessor, NgControl, Validators } from '@angular/forms';
import { ERROR_MESSAGES } from './error-messages.token';

let nextId = 0;

@Directive()
export abstract class BaseControl<T = string> implements ControlValueAccessor, OnInit {
  protected readonly ngControl = inject(NgControl, { self: true, optional: true });
  protected readonly cdr = inject(ChangeDetectorRef);
  protected readonly destroyRef = inject(DestroyRef);
  private readonly messages = inject(ERROR_MESSAGES);

  readonly label = input('');
  readonly hint = input('');
  readonly placeholder = input('');
  readonly readonly = input(false, { transform: booleanAttribute });

  readonly uid = `app-ctrl-${nextId++}`;
  readonly value = signal<T | null>(null);
  readonly disabled = signal(false);

  private onChange: (v: T | null) => void = () => {};
  private onTouched: () => void = () => {};

  constructor() {
    if (this.ngControl) this.ngControl.valueAccessor = this;
  }

  ngOnInit() {
    // رفع مشکل OnPush: با touched/reset/status از بیرون، UI به‌روز شود
    this.ngControl?.control?.events
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(() => this.cdr.markForCheck());
  }

  // ---- ControlValueAccessor ----
  writeValue(v: T | null): void { this.value.set(v); this.cdr.markForCheck(); }
  registerOnChange(fn: (v: T | null) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(d: boolean): void { this.disabled.set(d); this.cdr.markForCheck(); }

  // ---- متدهای کمکی برای فرزندان ----
  protected updateValue(v: T | null) { this.value.set(v); this.onChange(v); }
  protected markTouched() { this.onTouched(); }

  // ---- Validation state ----
  get control() { return this.ngControl?.control ?? null; }
  get isRequired(): boolean { return !!this.control?.hasValidator(Validators.required); }
  get isPending(): boolean { return !!this.control?.pending; }

  get showError(): boolean {
    const c = this.control;
    return !!c && c.invalid && (c.touched || c.dirty);
  }

  get errorMessage(): string {
    const errors = this.control?.errors;
    if (!errors) return '';
    const key = Object.keys(errors)[0];
    const fn = this.messages[key];
    return fn ? fn(errors[key], this.label()) : 'مقدار نامعتبر است';
  }
}
```

**نکات کلیدی:**

- `control.events` از Angular 18 به بعد موجود است. در نسخه‌های قدیمی‌تر از `merge(control.statusChanges, control.valueChanges)` استفاده کنید.
- کلاس فرزند فقط `updateValue()` و `markTouched()` را صدا می‌زند و با CVA کاری ندارد.
- `@Directive()` بدون selector لازم است تا Angular بتواند `input()`ها را از کلاس abstract بخواند.

## ۷. گام ۳: پوسته‌ی مشترک FieldShell

ظاهر label، hint و error را یک بار می‌نویسیم و همه‌ی کنترل‌ها از آن استفاده می‌کنند.

```typescript
// field-shell.component.ts
import { ChangeDetectionStrategy, Component, input } from '@angular/core';

@Component({
  selector: 'app-field-shell',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="field" [class.invalid]="showError()" [class.disabled]="disabled()">
      @if (label()) {
        <label [attr.for]="controlId()">
          {{ label() }}
          @if (required()) { <span class="req" aria-hidden="true">*</span> }
        </label>
      }

      <div class="control-wrap">
        <ng-content select="[prefix]" />
        <ng-content />
        <ng-content select="[suffix]" />
        @if (pending()) { <span class="spinner" aria-hidden="true"></span> }
      </div>

      @if (showError()) {
        <small class="error" [id]="controlId() + '-error'" role="alert">{{ error() }}</small>
      } @else if (hint()) {
        <small class="hint" [id]="controlId() + '-hint'">{{ hint() }}</small>
      }
    </div>
  `,
  styles: [`
    .field { display:flex; flex-direction:column; gap:.25rem; }
    .control-wrap { display:flex; align-items:center; gap:.5rem;
      border:1px solid #cbd5e1; border-radius:8px; padding:0 .75rem; }
    .field.invalid .control-wrap { border-color:#dc2626; }
    .field.disabled { opacity:.6; }
    .error { color:#dc2626; }
    .hint { color:#64748b; }
    .req { color:#dc2626; margin-inline-start:.15rem; }
  `],
})
export class FieldShellComponent {
  label = input('');
  hint = input('');
  error = input('');
  showError = input(false);
  required = input(false);
  pending = input(false);
  disabled = input(false);
  controlId = input.required<string>();
}
```

## ۸. گام ۴: ساخت app-input با ارث‌بری

```typescript
// app-input.component.ts
import { ChangeDetectionStrategy, Component, computed, input } from '@angular/core';
import { BaseControl } from '../base-control';
import { FieldShellComponent } from '../field-shell/field-shell.component';

@Component({
  selector: 'app-input',
  standalone: true,
  imports: [FieldShellComponent],
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: './app-input.component.html',
})
export class AppInputComponent extends BaseControl<string | number> {
  readonly type = input<'text' | 'email' | 'password' | 'number' | 'tel'>('text');
  readonly mask = input<'none' | 'thousand'>('none');
  readonly dir = input<'rtl' | 'ltr' | 'auto'>('auto');

  // مقدار نمایشی (ممکن است با مقدار واقعی فرق کند، مثلاً 1,250,000)
  readonly display = computed(() => {
    const v = this.value();
    if (v === null || v === undefined || v === '') return '';
    return this.mask() === 'thousand' ? this.formatThousand(String(v)) : String(v);
  });

  onInput(event: Event) {
    const el = event.target as HTMLInputElement;
    let raw = this.normalizeDigits(el.value);

    if (this.mask() === 'thousand') {
      raw = raw.replace(/[^\d]/g, '');
      el.value = this.formatThousand(raw);                 // نمایش فرمت‌شده
      this.updateValue(raw === '' ? null : Number(raw));   // مقدار خام به فرم
      return;
    }
    this.updateValue(raw === '' ? null : raw);
  }

  onBlur() { this.markTouched(); }

  private formatThousand(s: string) {
    return s.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  }
  /** تبدیل ارقام فارسی و عربی به انگلیسی */
  private normalizeDigits(s: string) {
    return s
      .replace(/[۰-۹]/g, d => String(d.charCodeAt(0) - 0x06f0))
      .replace(/[٠-٩]/g, d => String(d.charCodeAt(0) - 0x0660));
  }
}
```

```html
<!-- app-input.component.html -->
<app-field-shell
  [label]="label()" [hint]="hint()" [error]="errorMessage"
  [showError]="showError" [required]="isRequired" [pending]="isPending"
  [disabled]="disabled()" [controlId]="uid"
>
  <ng-content select="[prefix]" ngProjectAs="[prefix]" />

  <input
    [id]="uid"
    [type]="mask() === 'none' ? type() : 'text'"
    [attr.inputmode]="mask() === 'thousand' ? 'numeric' : null"
    [attr.dir]="dir() === 'auto' ? null : dir()"
    [placeholder]="placeholder()"
    [value]="display()"
    [disabled]="disabled()"
    [readOnly]="readonly()"
    [attr.aria-invalid]="showError"
    [attr.aria-required]="isRequired"
    [attr.aria-describedby]="showError ? uid + '-error' : (hint() ? uid + '-hint' : null)"
    (input)="onInput($event)"
    (blur)="onBlur()"
  />

  <ng-content select="[suffix]" ngProjectAs="[suffix]" />
</app-field-shell>
```

> **هشدار:** `mask="thousand"` هنگام تایپ در وسط عدد، کرسر را جابه‌جا می‌کند. برای production از `ngx-mask` یا یک directive اختصاصی با حفظ cursor استفاده کنید.

## ۹. گام ۵: استفاده در فرم

```typescript
form = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  amount: [null as number | null, [Validators.required, Validators.min(1000)]],
  username: ['', [Validators.required], [this.usernameTakenValidator]], // async
});

submit() {
  if (this.form.invalid) {
    this.form.markAllAsTouched(); // همه‌ی خطاها نمایش داده می‌شوند
    return;
  }
  console.log(this.form.getRawValue());
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <app-input label="ایمیل" type="email" formControlName="email" hint="مثال: a@b.com" />
  <app-input label="مبلغ (ریال)" mask="thousand" formControlName="amount" />
  <app-input label="نام کاربری" formControlName="username" />
  <button type="submit">ثبت</button>
</form>
```

## ۱۰. گام ۶: تست

```typescript
it('should show required error after touched', () => {
  const fixture = TestBed.createComponent(HostComponent);
  fixture.detectChanges();

  fixture.componentInstance.form.markAllAsTouched();
  fixture.detectChanges();

  expect(fixture.nativeElement.querySelector('.error')?.textContent)
    .toContain('الزامی');
});
```

## ۱۱. گام ۷: ساخت کنترل‌های بعدی

با وجود `BaseControl` و `FieldShell`، ساخت `app-select` بسیار کوتاه است:

```typescript
@Component({
  selector: 'app-select',
  standalone: true,
  imports: [FieldShellComponent],
  template: `
    <app-field-shell [label]="label()" [error]="errorMessage" [showError]="showError"
                     [required]="isRequired" [controlId]="uid">
      <select [id]="uid" [disabled]="disabled()"
              (change)="updateValue($any($event.target).value)" (blur)="markTouched()">
        @for (o of options(); track o.value) {
          <option [value]="o.value" [selected]="o.value === value()">{{ o.label }}</option>
        }
      </select>
    </app-field-shell>
  `,
})
export class AppSelectComponent extends BaseControl<string> {
  options = input<{ value: string; label: string }[]>([]);
}
```

## ۱۲. نکات مهم و خطاهای رایج

- **Circular dependency:** هم‌زمان `NG_VALUE_ACCESSOR` provide نکنید و `NgControl` را inject کنید.
- **OnPush و reset():** بدون subscribe به `control.events` (یا `statusChanges`/`valueChanges`) و `markForCheck()`، پس از `markAllAsTouched()` خطاها نمایش داده نمی‌شوند.
- **مقدار خام در برابر مقدار نمایشی:** در mask همیشه مقدار خام (عدد بدون کاما) را به فرم بدهید، نه رشته‌ی فرمت‌شده.
- **ارقام فارسی:** قبل از ارسال به فرم، ارقام فارسی/عربی را به انگلیسی تبدیل کنید.
- **ولیدیتور داخلی (اختیاری):** با پیاده‌سازی `Validator` و provide کردن `NG_VALIDATORS` می‌توانید validation را داخل خود کامپوننت نگه دارید.
- **پروژه‌های بزرگ:** پیام‌ها را از طریق `ERROR_MESSAGES` مدیریت کنید و برای هر کنترل یک Story در Storybook بسازید.

## ۱۳. قابلیت‌های قابل افزودن

| قابلیت | روش پیشنهادی |
| --- | --- |
| Cursor-safe mask | directive جدا یا `ngx-mask` |
| تقویم شمسی | Datepicker با `Intl` یا `jalali-moment` |
| i18n پیام‌ها | override کردن `ERROR_MESSAGES` با `useFactory` و سرویس ترجمه |
| Debounce برای async validator | `updateOn: 'blur'` یا `timer` در validator |
| Storybook | یک story برای هر state: normal / error / disabled / pending |
| Testing | `ComponentHarness` برای تست کنترل‌ها |

## ۱۴. چک‌لیست نهایی

- [ ] `ERROR_MESSAGES` ساخته و در صورت نیاز override شده است
- [ ] `BaseControl` با `NgControl` (`self`, `optional`) پیاده شده است
- [ ] `FieldShell` برای label / hint / error مشترک است
- [ ] هر کنترل از `BaseControl` ارث می‌برد و فقط `updateValue` و `markTouched` را صدا می‌زند
- [ ] ویژگی‌های ARIA روی المان‌ها تنظیم شده است
- [ ] `markAllAsTouched()` و `reset()` با OnPush تست شده است
- [ ] تست و Storybook برای هر state نوشته شده است

**پیشنهاد برای پورتفولیو:** این زیرساخت را به‌صورت یک کتابخانه‌ی مستقل با `ng-packagr` و Storybook منتشر کنید و در README آن، همین مستند را به‌عنوان راهنما قرار دهید.

**گام‌های بعدی پیشنهادی:** Datepicker شمسی و directive mask با حفظ cursor بر پایه‌ی همین `BaseControl`.
