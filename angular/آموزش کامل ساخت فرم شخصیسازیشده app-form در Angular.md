# آموزش کامل ساخت فرم شخصی‌سازی‌شده `<app-form>` در Angular

نسخه اصلاح‌شده و کامل · Angular 18+ · Standalone · Signal Inputs · Reactive Forms

## ۰. این آموزش چه چیزی می‌سازد؟

هدف این است که به‌جای نوشتن HTML تکراری برای هر فرم، یک کامپوننت هوشمند داشته باشیم که فقط یک **config** بگیرد و فرم کامل را بسازد:

```html
<app-form [fields]="fields" [initialValue]="model()" (submitted)="save($event)" />
```

خروجی نهایی این آموزش:

- ساخت خودکار `FormGroup` / `FormArray` تو در تو از روی config
- کنترل‌های سفارشی با `ControlValueAccessor` (Text, Textarea, Select, Checkbox, Radio)
- Validator های Sync، Async و Cross-field (مثل کد ملی ایران و یکتا بودن ایمیل)
- پیام خطای مرکزی و قابل تغییر در کل پروژه
- فیلد شرطی، FormArray پویا، حالت Edit، Reset
- نمایش خطاهای Backend (.NET) روی فیلدها
- Layout گرید ۱۲ ستونی، استایل RTL، دسترس‌پذیری
- تست‌های واحد

> این نسخه چند باگ و محدودیت نسخه قبلی را اصلاح کرده است: تداخل نام `onChange`، sync نشدن FormArray های تو در تو، پاک نشدن آرایه در Reset، شرطی بودن فقط فیلدهای سطح اول، و ذخیره رمز عبور در Autosave.

## ۱. معماری

```
Page ──(fields config)──▶ <app-form>
                              │  ساخت فرم با FormFactory
                              ▼
                         <app-field>   (بر اساس type کنترل مناسب را انتخاب می‌کند؛ بازگشتی برای group/array)
                              ├─▶ <app-text-input>  ┐
                              ├─▶ <app-select>      │ همه از BaseControl
                              ├─▶ <app-checkbox>    │ ارث می‌برند و
                              ├─▶ <app-radio>       ┘ ControlValueAccessor هستند
                              └─▶ <app-field-errors>
```

چهار اصل طراحی:

1. **Config جدا از UI**: صفحه فقط توصیف می‌کند چه فیلدهایی می‌خواهد.
2. **Factory جدا از کامپوننت**: ساخت کنترل‌ها در یک سرویس قابل تست است.
3. **کنترل‌ها = CVA**: هر کنترل جدید را می‌توان در فرم‌های دستی هم استفاده کرد.
4. **پیام‌ها مرکزی**: با یک `InjectionToken` برای کل پروژه (و i18n) عوض می‌شوند.

## ۲. ساختار پوشه‌ها

```
src/app/shared/forms/
├── models/field-config.ts
├── tokens/validation-messages.token.ts
├── validators/custom-validators.ts
├── validators/custom-validators.spec.ts
├── utils/server-errors.ts
├── services/form-factory.service.ts
├── services/form-factory.service.spec.ts
├── controls/
│   ├── base-control.ts
│   ├── text-input.component.ts
│   ├── select.component.ts
│   ├── checkbox.component.ts
│   └── radio.component.ts
├── field-errors.component.ts
├── field.component.ts
├── app-form.component.ts
└── index.ts            // Public API
```

در `tsconfig.json` یک alias بگذارید تا import ها تمیز شوند: `"paths": { "@app/*": ["src/app/*"] }`.

## ۳. مدل Config

همه چیز از این interface شروع می‌شود. اگر قابلیتی لازم شد (مثلاً `hint` یا `mask`)، اول اینجا اضافه می‌شود.

```ts
// models/field-config.ts
import { AsyncValidatorFn, FormGroup, ValidatorFn } from '@angular/forms';

export type FieldType =
  | 'text' | 'email' | 'password' | 'number' | 'tel' | 'date'
  | 'textarea' | 'select' | 'radio' | 'checkbox'
  | 'group' | 'array';

export interface SelectOption { value: string | number; label: string; }

export interface FieldConfig {
  key: string;                      // نام کنترل در FormGroup
  label: string;
  type: FieldType;
  placeholder?: string;
  value?: unknown;                  // مقدار اولیه
  disabled?: boolean;
  options?: SelectOption[];         // select و radio
  validators?: ValidatorFn[];
  asyncValidators?: AsyncValidatorFn[];
  updateOn?: 'change' | 'blur';     // برای کنترل‌های ساده
  messages?: Record<string, string>;                // override پیام خطا برای همین فیلد
  visibleWhen?: (root: FormGroup) => boolean;       // فیلد شرطی (در همه سطوح)
  colSpan?: number;                 // 1 تا 12 (پیش‌فرض 12)
  children?: FieldConfig[];         // برای group و array
  maxItems?: number;                // برای array
}
```

نکته: `updateOn: 'submit'` را عمداً حذف کردیم. در فرم‌های config-driven بهتر است آن را روی کل فرم (در سطح `FormGroup`) تنظیم کنید، نه روی تک‌تک فیلدها.

## ۴. پیام‌های خطا (مرکزی)

```ts
// tokens/validation-messages.token.ts
import { InjectionToken } from '@angular/core';

export type MessageMap = Record<string, (err: any) => string>;

export const DEFAULT_MESSAGES: MessageMap = {
  required: () => 'این فیلد الزامی است',
  email: () => 'ایمیل معتبر نیست',
  minlength: e => `حداقل ${e.requiredLength} کاراکتر وارد کنید`,
  maxlength: e => `حداکثر ${e.requiredLength} کاراکتر مجاز است`,
  min: e => `حداقل مقدار ${e.min} است`,
  max: e => `حداکثر مقدار ${e.max} است`,
  pattern: () => 'فرمت وارد شده صحیح نیست',
  nationalCode: () => 'کد ملی معتبر نیست',
  passwordMismatch: () => 'رمز عبور و تکرار آن یکسان نیستند',
  emailTaken: () => 'این ایمیل قبلاً ثبت شده است',
};

export const VALIDATION_MESSAGES = new InjectionToken<MessageMap>('VALIDATION_MESSAGES', {
  providedIn: 'root',
  factory: () => DEFAULT_MESSAGES,
});
```

برای شخصی‌سازی یا i18n در `app.config.ts`:

```ts
{ provide: VALIDATION_MESSAGES, useValue: { ...DEFAULT_MESSAGES, required: () => 'پر کردن این فیلد ضروری است' } }
```

## ۵. Validator های سفارشی

سه نوع validator داریم و هر کدام یک کاربرد:

| نوع | روی چه چیزی | مثال |
| --- | --- | --- |
| Sync | یک کنترل | کد ملی |
| Cross-field | کل `FormGroup` | تطابق رمز و تکرار آن |
| Async | یک کنترل، با API | یکتا بودن ایمیل |

```ts
// validators/custom-validators.ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors, ValidatorFn } from '@angular/forms';
import { Observable, catchError, map, of, switchMap, timer } from 'rxjs';

// 1) Sync: کد ملی ایران
export function nationalCode(): ValidatorFn {
  return (c: AbstractControl): ValidationErrors | null => {
    const v = String(c.value ?? '');
    if (!v) return null;                                  // خالی بودن با required چک می‌شود
    if (!/^\d{10}$/.test(v) || /^(\d)\1{9}$/.test(v)) return { nationalCode: true };
    const check = +v[9];
    const sum = v.slice(0, 9).split('').reduce((s, d, i) => s + +d * (10 - i), 0) % 11;
    return (sum < 2 ? check === sum : check === 11 - sum) ? null : { nationalCode: true };
  };
}

// 2) Cross-field: روی FormGroup اعمال می‌شود
export const passwordMatch: ValidatorFn = (g: AbstractControl) => {
  const p = g.get('password')?.value;
  const c = g.get('confirmPassword')?.value;
  return p && c && p !== c ? { passwordMismatch: true } : null;
};

// 3) Async با debounce
export function uniqueEmail(exists: (email: string) => Observable<boolean>): AsyncValidatorFn {
  return c => {
    if (!c.value) return of(null);
    return timer(400).pipe(                               // با تایپ جدید، subscription قبلی لغو می‌شود
      switchMap(() => exists(c.value)),
      map(taken => (taken ? { emailTaken: true } : null)),
      catchError(() => of(null)),                         // خطای شبکه نباید فرم را قفل کند
    );
  };
}
```

## ۶. FormFactory: ساخت فرم از روی Config

این سرویس سه کار می‌کند: ساخت فرم، پر کردن فرم در حالت Edit (با sync کردن آیتم‌های آرایه به‌صورت **بازگشتی**) و Reset.

```ts
// services/form-factory.service.ts
import { Injectable, inject } from '@angular/core';
import { AbstractControl, FormArray, FormGroup, NonNullableFormBuilder, ValidatorFn } from '@angular/forms';
import { FieldConfig, FieldType } from '../models/field-config';

@Injectable({ providedIn: 'root' })
export class FormFactory {
  private fb = inject(NonNullableFormBuilder);

  build(fields: FieldConfig[], groupValidators: ValidatorFn[] = []): FormGroup {
    const controls: Record<string, AbstractControl> = {};
    for (const f of fields) controls[f.key] = this.createControl(f);
    return this.fb.group(controls, { validators: groupValidators });
  }

  createControl(f: FieldConfig): AbstractControl {
    switch (f.type) {
      case 'group': return this.build(f.children ?? []);
      case 'array': return this.fb.array<AbstractControl>([], { validators: f.validators });
      default:
        return this.fb.control<unknown>(
          { value: f.value ?? this.defaultOf(f.type), disabled: !!f.disabled },
          { validators: f.validators, asyncValidators: f.asyncValidators, updateOn: f.updateOn ?? 'change' },
        );
    }
  }

  createArrayItem(f: FieldConfig): FormGroup {
    return this.build(f.children ?? []);
  }

  /** حالت Edit: اول تعداد آیتم‌های آرایه‌ها را می‌سازد، بعد مقدار را patch می‌کند */
  patch(form: FormGroup, fields: FieldConfig[], value: Record<string, any> | null): void {
    this.syncArrays(form, fields, value ?? {});
    form.patchValue(value ?? {});
    form.markAsPristine();
    form.markAsUntouched();
  }

  /** برگرداندن فرم به حالت اولیه (آرایه‌ها هم پاک یا دوباره ساخته می‌شوند) */
  reset(form: FormGroup, fields: FieldConfig[], initial: Record<string, any> | null): void {
    this.syncArrays(form, fields, initial ?? {});
    initial ? form.patchValue(initial) : form.reset();
    form.markAsPristine();
    form.markAsUntouched();
  }

  // بازگشتی: group داخل group، array داخل group، و حتی array داخل آیتم array
  private syncArrays(group: FormGroup, fields: FieldConfig[], value: Record<string, any>): void {
    for (const f of fields) {
      if (f.type === 'array') {
        const arr = group.get(f.key) as FormArray;
        arr.clear();
        for (const itemValue of (value?.[f.key] ?? []) as Record<string, any>[]) {
          const item = this.createArrayItem(f);
          this.syncArrays(item, f.children ?? [], itemValue);
          arr.push(item);
        }
      } else if (f.type === 'group') {
        this.syncArrays(group.get(f.key) as FormGroup, f.children ?? [], value?.[f.key] ?? {});
      }
    }
  }

  private defaultOf(t: FieldType) {
    return t === 'checkbox' ? false : t === 'number' ? null : '';
  }
}
```

چرا `patchValue` به‌تنهایی کافی نیست؟ چون `FormArray` خالی است و `patchValue` برای آیتمی که وجود ندارد کنترل نمی‌سازد؛ باید اول تعداد آیتم‌ها ساخته شود.

## ۷. BaseControl: کلاس پایه کنترل‌های سفارشی

این کلاس `ControlValueAccessor` را یک بار پیاده می‌کند. به‌جای provide کردن `NG_VALUE_ACCESSOR`، خودمان را روی `NgControl` ثبت می‌کنیم؛ این روش وابستگی حلقوی (circular DI) ایجاد نمی‌کند و دسترسی به خطاها و `touched/dirty` را هم می‌دهد.

```ts
// controls/base-control.ts
import { Directive, inject, signal } from '@angular/core';
import { ControlValueAccessor, NgControl, Validators } from '@angular/forms';

let nextId = 0;

@Directive()
export abstract class BaseControl<T> implements ControlValueAccessor {
  readonly ngControl = inject(NgControl, { self: true, optional: true });
  readonly id = `ctrl-${nextId++}`;

  value = signal<T | null>(null);
  disabled = signal(false);

  // اصلاح باگ: نام‌های خصوصی با _ شروع می‌شوند تا با متدهای کلاس‌های فرزند تداخل نکنند
  private _onChange: (v: T | null) => void = () => {};
  private _onTouched: () => void = () => {};

  constructor() {
    if (this.ngControl) this.ngControl.valueAccessor = this;
  }

  get control() { return this.ngControl?.control ?? null; }
  get showError() { const c = this.control; return !!c && c.invalid && (c.touched || c.dirty); }
  get isRequired() {
    const c = this.control;
    return !!c && (c.hasValidator(Validators.required) || c.hasValidator(Validators.requiredTrue));
  }

  // --- ControlValueAccessor ---
  writeValue(v: T | null) { this.value.set(v); }
  registerOnChange(fn: (v: T | null) => void) { this._onChange = fn; }
  registerOnTouched(fn: () => void) { this._onTouched = fn; }
  setDisabledState(d: boolean) { this.disabled.set(d); }

  protected update(v: T | null) { this.value.set(v); this._onChange(v); }
  touch() { this._onTouched(); }
}
```

> برای ساخت هر کنترل جدید (DatePicker شمسی، Autocomplete، Upload) فقط از `BaseControl` ارث‌بری کنید و `update(مقدار)` را صدا بزنید.

## ۸. نمایش خطاها

```ts
// field-errors.component.ts
import { Component, inject, input } from '@angular/core';
import { AbstractControl } from '@angular/forms';
import { VALIDATION_MESSAGES } from './tokens/validation-messages.token';

@Component({
  selector: 'app-field-errors',
  standalone: true,
  template: `@if (message) { <small class="error" role="alert">{{ message }}</small> }`,
})
export class FieldErrorsComponent {
  control = input<AbstractControl | null>(null);
  messages = input<Record<string, string>>({});
  private global = inject(VALIDATION_MESSAGES);

  get message(): string | null {
    const c = this.control();
    if (!c || !c.errors || !(c.touched || c.dirty)) return null;
    const [key, val] = Object.entries(c.errors)[0];
    if (key === 'server') return String(val);            // خطای برگشتی از Backend
    return this.messages()[key] ?? this.global[key]?.(val) ?? 'مقدار نامعتبر است';
  }
}
```

ترتیب اولویت پیام: ۱) `messages` همان فیلد ← ۲) پیام مرکزی token ← ۳) پیام پیش‌فرض.

## ۹. کنترل‌های سفارشی

### Text / Textarea

```ts
// controls/text-input.component.ts
import { Component, input } from '@angular/core';
import { BaseControl } from './base-control';
import { FieldErrorsComponent } from '../field-errors.component';

@Component({
  selector: 'app-text-input',
  standalone: true,
  imports: [FieldErrorsComponent],
  template: `
    <div class="field" [class.has-error]="showError">
      <label [for]="id">{{ label() }} @if (isRequired) { <span class="req">*</span> }</label>

      @if (multiline()) {
        <textarea [id]="id" rows="4" [value]="value() ?? ''" [placeholder]="placeholder()"
          [disabled]="disabled()" [attr.aria-invalid]="showError"
          (input)="onInput($event)" (blur)="touch()"></textarea>
      } @else {
        <input [id]="id" [type]="type()" [value]="value() ?? ''" [placeholder]="placeholder()"
          [disabled]="disabled()" [attr.aria-invalid]="showError"
          (input)="onInput($event)" (blur)="touch()" />
      }

      @if (control?.pending) { <small class="hint">در حال بررسی...</small> }
      <app-field-errors [control]="control" [messages]="messages()" />
    </div>
  `,
})
export class TextInputComponent extends BaseControl<string | number> {
  label = input.required<string>();
  type = input('text');
  placeholder = input('');
  multiline = input(false);
  messages = input<Record<string, string>>({});

  onInput(e: Event) {
    const el = e.target as HTMLInputElement;
    this.update(this.type() === 'number' ? (el.value === '' ? null : +el.value) : el.value);
  }
}
```

### Select (با اصلاح نام متد)

```ts
// controls/select.component.ts
import { Component, input } from '@angular/core';
import { BaseControl } from './base-control';
import { FieldErrorsComponent } from '../field-errors.component';
import { SelectOption } from '../models/field-config';

@Component({
  selector: 'app-select',
  standalone: true,
  imports: [FieldErrorsComponent],
  template: `
    <div class="field" [class.has-error]="showError">
      <label [for]="id">{{ label() }} @if (isRequired) { <span class="req">*</span> }</label>
      <select [id]="id" [disabled]="disabled()" [attr.aria-invalid]="showError"
              (change)="onSelect($event)" (blur)="touch()">
        <option value="" [selected]="value() === '' || value() == null">انتخاب کنید...</option>
        @for (o of options(); track o.value) {
          <option [value]="o.value" [selected]="o.value === value()">{{ o.label }}</option>
        }
      </select>
      <app-field-errors [control]="control" [messages]="messages()" />
    </div>
  `,
})
export class SelectComponent extends BaseControl<string | number> {
  label = input.required<string>();
  options = input<SelectOption[]>([]);
  messages = input<Record<string, string>>({});

  onSelect(e: Event) {
    const raw = (e.target as HTMLSelectElement).value;
    const opt = this.options().find(o => String(o.value) === raw);
    this.update(opt ? opt.value : '');           // نوع اصلی (number/string) حفظ می‌شود
  }
}
```

### Checkbox

```ts
// controls/checkbox.component.ts
import { Component, input } from '@angular/core';
import { BaseControl } from './base-control';
import { FieldErrorsComponent } from '../field-errors.component';

@Component({
  selector: 'app-checkbox',
  standalone: true,
  imports: [FieldErrorsComponent],
  template: `
    <div class="field checkbox" [class.has-error]="showError">
      <label>
        <input type="checkbox" [checked]="!!value()" [disabled]="disabled()" [attr.aria-invalid]="showError"
               (change)="update($any($event.target).checked)" (blur)="touch()" />
        {{ label() }} @if (isRequired) { <span class="req">*</span> }
      </label>
      <app-field-errors [control]="control" [messages]="messages()" />
    </div>
  `,
})
export class CheckboxComponent extends BaseControl<boolean> {
  label = input.required<string>();
  messages = input<Record<string, string>>({});
}
```

### Radio (جدید)

```ts
// controls/radio.component.ts
import { Component, input } from '@angular/core';
import { BaseControl } from './base-control';
import { FieldErrorsComponent } from '../field-errors.component';
import { SelectOption } from '../models/field-config';

@Component({
  selector: 'app-radio',
  standalone: true,
  imports: [FieldErrorsComponent],
  template: `
    <fieldset class="field radio-group" [class.has-error]="showError">
      <legend>{{ label() }} @if (isRequired) { <span class="req">*</span> }</legend>
      @for (o of options(); track o.value) {
        <label class="radio">
          <input type="radio" [name]="id" [checked]="o.value === value()" [disabled]="disabled()"
                 (change)="update(o.value)" (blur)="touch()" />
          {{ o.label }}
        </label>
      }
      <app-field-errors [control]="control" [messages]="messages()" />
    </fieldset>
  `,
})
export class RadioComponent extends BaseControl<string | number> {
  label = input.required<string>();
  options = input<SelectOption[]>([]);
  messages = input<Record<string, string>>({});
}
```

## ۱۰. `<app-field>`: انتخاب کنترل بر اساس type

این کامپوننت بازگشتی است: برای `group` و `array` خودش را دوباره برای فرزندان صدا می‌زند. همچنین `colSpan` را به گرید می‌دهد و فیلدهای شرطی مخفی را نشان نمی‌دهد.

```ts
// field.component.ts
import { Component, inject, input } from '@angular/core';
import { AbstractControl, FormArray, FormControl, FormGroup, ReactiveFormsModule } from '@angular/forms';
import { FieldConfig } from './models/field-config';
import { FormFactory } from './services/form-factory.service';
import { TextInputComponent } from './controls/text-input.component';
import { SelectComponent } from './controls/select.component';
import { CheckboxComponent } from './controls/checkbox.component';
import { RadioComponent } from './controls/radio.component';
import { FieldErrorsComponent } from './field-errors.component';

@Component({
  selector: 'app-field',
  standalone: true,
  imports: [ReactiveFormsModule, TextInputComponent, SelectComponent, CheckboxComponent, RadioComponent, FieldErrorsComponent],
  host: { '[style.grid-column]': "'span ' + (config().colSpan ?? 12)" },
  template: `
    @if (visible()) {
      @switch (config().type) {
        @case ('select') {
          <app-select [formControl]="asControl()" [label]="config().label"
                      [options]="config().options ?? []" [messages]="config().messages ?? {}" />
        }
        @case ('radio') {
          <app-radio [formControl]="asControl()" [label]="config().label"
                     [options]="config().options ?? []" [messages]="config().messages ?? {}" />
        }
        @case ('checkbox') {
          <app-checkbox [formControl]="asControl()" [label]="config().label" [messages]="config().messages ?? {}" />
        }
        @case ('group') {
          <fieldset class="group">
            <legend>{{ config().label }}</legend>
            <div class="grid">
              @for (child of config().children; track child.key) {
                <app-field [config]="child" [control]="asGroup().get(child.key)!" />
              }
            </div>
          </fieldset>
        }
        @case ('array') {
          <fieldset class="group">
            <legend>{{ config().label }}</legend>
            @for (item of asArray().controls; track item; let i = $index) {
              <div class="array-row grid">
                @for (child of config().children; track child.key) {
                  <app-field [config]="child" [control]="$any(item).get(child.key)!" />
                }
                <button type="button" class="btn-icon" (click)="remove(i)" aria-label="حذف">🗑</button>
              </div>
            }
            <button type="button" class="btn-secondary" (click)="add()"
                    [disabled]="asArray().length >= (config().maxItems ?? 10)">+ افزودن</button>
            <app-field-errors [control]="control()" />
          </fieldset>
        }
        @default {
          <app-text-input [formControl]="asControl()" [label]="config().label"
                          [type]="config().type === 'textarea' ? 'text' : config().type"
                          [multiline]="config().type === 'textarea'"
                          [placeholder]="config().placeholder ?? ''" [messages]="config().messages ?? {}" />
        }
      }
    }
  `,
})
export class FieldComponent {
  config = input.required<FieldConfig>();
  control = input.required<AbstractControl>();
  private factory = inject(FormFactory);

  asControl() { return this.control() as FormControl; }
  asGroup() { return this.control() as FormGroup; }
  asArray() { return this.control() as FormArray; }

  // فیلد شرطی مخفی، توسط AppForm «disable» می‌شود؛ پس اینجا کافی است وضعیت را بخوانیم
  visible() { return !this.config().visibleWhen || this.control().enabled; }

  add() { this.asArray().push(this.factory.createArrayItem(this.config())); this.asArray().markAsDirty(); }
  remove(i: number) { this.asArray().removeAt(i); this.asArray().markAsDirty(); }
}
```

توجه: فیلدی که هم `visibleWhen` دارد و هم `disabled: true` است، مخفی دیده می‌شود. برای فیلد «فقط‌خواندنی ولی شرطی» از `disabled` استفاده نکنید.

## ۱۱. کامپوننت اصلی `<app-form>`

```ts
// app-form.component.ts
import { Component, ElementRef, computed, effect, inject, input, output, untracked } from '@angular/core';
import { FormGroup, ReactiveFormsModule, ValidatorFn } from '@angular/forms';
import { filter, startWith, take } from 'rxjs';
import { FieldConfig } from './models/field-config';
import { FormFactory } from './services/form-factory.service';
import { FieldComponent } from './field.component';
import { FieldErrorsComponent } from './field-errors.component';

@Component({
  selector: 'app-form',
  standalone: true,
  imports: [ReactiveFormsModule, FieldComponent, FieldErrorsComponent],
  template: `
    <form [formGroup]="form()" (ngSubmit)="submit()" novalidate class="app-form">
      <div class="grid">
        @for (f of fields(); track f.key) {
          <app-field [config]="f" [control]="form().get(f.key)!" />
        }
      </div>

      <!-- خطای سطح فرم (مثل passwordMismatch) -->
      <app-field-errors [control]="form()" />

      <div class="actions">
        <button type="submit" class="btn-primary" [disabled]="loading() || form().pending">
          {{ loading() ? 'در حال ارسال...' : submitLabel() }}
        </button>
        <button type="button" class="btn-secondary" (click)="reset()">پاک کردن</button>
        @if (showCancel()) { <button type="button" class="btn-secondary" (click)="cancelled.emit()">انصراف</button> }
      </div>
    </form>
  `,
})
export class AppFormComponent {
  // ---- Inputs / Outputs ----
  fields = input.required<FieldConfig[]>();
  groupValidators = input<ValidatorFn[]>([]);
  initialValue = input<Record<string, any> | null>(null);    // حالت Edit
  submitLabel = input('ثبت');
  loading = input(false);
  showCancel = input(false);
  includeDisabled = input(false);       // true → getRawValue ، false → value

  submitted = output<Record<string, any>>();
  cancelled = output<void>();
  valueChange = output<Record<string, any>>();

  // ---- State ----
  private factory = inject(FormFactory);
  private host = inject(ElementRef<HTMLElement>);

  form = computed<FormGroup>(() => {
    const form = this.factory.build(this.fields(), this.groupValidators());
    this.applyVisibility(form, this.fields());          // وضعیت اولیه فیلدهای شرطی قبل از اولین رندر
    return form;
  });

  constructor() {
    // حالت Edit: مقدار اولیه (شامل ساخت آیتم‌های FormArray)
    effect(() => {
      const form = this.form();
      const value = this.initialValue();
      if (value) untracked(() => this.factory.patch(form, this.fields(), value));
    });

    // فیلدهای شرطی + انتشار تغییرات
    effect(onCleanup => {
      const form = this.form();
      const sub = form.valueChanges.subscribe(v => {
        this.applyVisibility(form, this.fields());
        this.valueChange.emit(form.getRawValue());
      });
      onCleanup(() => sub.unsubscribe());
    });
  }

  /** بازگشتی: فیلد مخفی disable می‌شود تا validation آن اجرا نشود و در value نیاید */
  private applyVisibility(root: FormGroup, fields: FieldConfig[], parent: FormGroup = root): void {
    for (const f of fields) {
      const ctrl = parent.get(f.key);
      if (!ctrl) continue;
      if (f.visibleWhen) {
        const show = f.visibleWhen(root);
        if (show && ctrl.disabled) ctrl.enable({ emitEvent: false });
        if (!show && ctrl.enabled) ctrl.disable({ emitEvent: false });
      }
      if (f.type === 'group' && ctrl instanceof FormGroup && ctrl.enabled) {
        this.applyVisibility(root, f.children ?? [], ctrl);
      }
    }
  }

  submit() {
    const form = this.form();
    form.markAllAsTouched();

    // اگر Async validator در حال اجراست، صبر کن تا تمام شود
    form.statusChanges.pipe(startWith(form.status), filter(s => s !== 'PENDING'), take(1)).subscribe(() => {
      if (form.invalid) { this.focusFirstInvalid(); return; }
      this.submitted.emit(this.includeDisabled() ? form.getRawValue() : form.value);
    });
  }

  reset() {
    this.factory.reset(this.form(), this.fields(), this.initialValue());
    this.applyVisibility(this.form(), this.fields());
  }

  private focusFirstInvalid() {
    (this.host.nativeElement as HTMLElement)
      .querySelector<HTMLElement>('[aria-invalid="true"]')?.focus();
  }
}
```

نکته: `form.value` فیلدهای disabled (از جمله فیلد شرطی مخفی) را شامل نمی‌شود و معمولاً همین را می‌خواهیم؛ اگر API شما همه چیز را می‌خواهد `includeDisabled` را true کنید.

## ۱۲. Public API و تنظیمات پروژه

```ts
// index.ts
export * from './app-form.component';
export * from './models/field-config';
export * from './validators/custom-validators';
export * from './utils/server-errors';
export * from './tokens/validation-messages.token';
```

```ts
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { DEFAULT_MESSAGES, VALIDATION_MESSAGES } from '@app/shared/forms';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    { provide: VALIDATION_MESSAGES, useValue: { ...DEFAULT_MESSAGES } },   // اینجا پیام‌ها را شخصی‌سازی کنید
  ],
};
```

## ۱۳. استایل (RTL + گرید ۱۲ ستونی)

در `styles.scss` سراسری قرار دهید. از **Logical Properties** استفاده شده تا فرم هم در RTL و هم LTR درست کار کند. در `index.html` هم `<html lang="fa" dir="rtl">` بگذارید.

```scss
:root {
  --c-primary: #2563eb;
  --c-error: #dc2626;
  --c-border: #cbd5e1;
  --c-text: #0f172a;
  --c-muted: #64748b;
  --radius: 8px;
  --gap: 16px;
}

.app-form .grid, .group .grid { display: grid; grid-template-columns: repeat(12, 1fr); gap: var(--gap); }

/* موبایل: همه فیلدها تمام‌عرض */
@media (max-width: 640px) { .grid > * { grid-column: span 12 !important; } }

.field { display: flex; flex-direction: column; gap: 6px; }
.field label, .field legend { font-weight: 600; color: var(--c-text); }
.field .req { color: var(--c-error); margin-inline-start: 4px; }

.field input:not([type=checkbox]):not([type=radio]), .field select, .field textarea {
  padding-block: 10px; padding-inline: 12px;
  border: 1px solid var(--c-border); border-radius: var(--radius);
  font: inherit; background: #fff;
}
.field input:focus, .field select:focus, .field textarea:focus {
  outline: 2px solid var(--c-primary); outline-offset: 1px;
}
.field.has-error input, .field.has-error select, .field.has-error textarea { border-color: var(--c-error); }
.field .error { color: var(--c-error); font-size: .85rem; }
.field .hint  { color: var(--c-muted); font-size: .85rem; }
.field.checkbox label, .field .radio { display: flex; gap: 8px; align-items: center; font-weight: 400; }

fieldset.group { border: 1px solid var(--c-border); border-radius: var(--radius); padding: var(--gap); margin: 0; }
fieldset.group > legend { padding-inline: 8px; font-weight: 700; }
.array-row { align-items: end; margin-block-end: var(--gap); }

.actions { display: flex; gap: 12px; margin-block-start: calc(var(--gap) * 1.5); }
button { font: inherit; cursor: pointer; border-radius: var(--radius); padding: 10px 18px; border: 1px solid transparent; }
button:disabled { opacity: .55; cursor: not-allowed; }
.btn-primary { background: var(--c-primary); color: #fff; }
.btn-secondary { background: #fff; border-color: var(--c-border); }
.btn-icon { background: transparent; border: 0; padding: 6px; }
```

## ۱۴. استفاده در یک صفحه (مثال کامل ثبت‌نام)

این مثال همه قابلیت‌ها را نشان می‌دهد: Validator های sync/async، cross-field، فیلد شرطی، گروه تو در تو، آرایه پویا، `colSpan` و خطای سرور.

```ts
// register.page.ts
import { Component, inject, signal, viewChild } from '@angular/core';
import { Validators } from '@angular/forms';
import { HttpErrorResponse } from '@angular/common/http';
import { AppFormComponent, FieldConfig, applyServerErrors, nationalCode, passwordMatch, uniqueEmail } from '@app/shared/forms';
import { UserApi } from './user.api';

@Component({
  standalone: true,
  imports: [AppFormComponent],
  template: `
    <app-form #f
      [fields]="fields"
      [groupValidators]="groupValidators"
      [initialValue]="model()"
      [loading]="saving()"
      submitLabel="ثبت‌نام"
      (submitted)="save($event)" />
  `,
})
export class RegisterPage {
  private api = inject(UserApi);
  private formCmp = viewChild.required<AppFormComponent>('f');

  saving = signal(false);
  model = signal<Record<string, any> | null>(null);   // در حالت Edit از API پر کنید
  groupValidators = [passwordMatch];

  // مهم: fields باید reference ثابت باشد (property کلاس، نه getter و نه ساخته‌شده داخل template)
  fields: FieldConfig[] = [
    { key: 'fullName', label: 'نام و نام خانوادگی', type: 'text', colSpan: 6,
      validators: [Validators.required, Validators.minLength(3)] },

    { key: 'nationalCode', label: 'کد ملی', type: 'text', colSpan: 6,
      validators: [Validators.required, nationalCode()] },

    { key: 'email', label: 'ایمیل', type: 'email', updateOn: 'blur',
      validators: [Validators.required, Validators.email],
      asyncValidators: [uniqueEmail(e => this.api.emailExists(e))] },

    { key: 'password', label: 'رمز عبور', type: 'password', colSpan: 6,
      validators: [Validators.required, Validators.minLength(8), Validators.pattern(/^(?=.*[A-Za-z])(?=.*\d).+$/)],
      messages: { pattern: 'رمز باید ترکیبی از حروف و عدد باشد' } },

    { key: 'confirmPassword', label: 'تکرار رمز عبور', type: 'password', colSpan: 6,
      validators: [Validators.required] },

    { key: 'role', label: 'نقش', type: 'radio', validators: [Validators.required],
      options: [{ value: 'user', label: 'کاربر' }, { value: 'company', label: 'شرکت' }] },

    { key: 'companyName', label: 'نام شرکت', type: 'text', validators: [Validators.required],
      visibleWhen: root => root.get('role')?.value === 'company' },       // فیلد شرطی

    { key: 'address', label: 'آدرس', type: 'group', children: [
        { key: 'city', label: 'شهر', type: 'text', colSpan: 6, validators: [Validators.required] },
        { key: 'zip', label: 'کد پستی', type: 'text', colSpan: 6, validators: [Validators.pattern(/^\d{10}$/)] },
    ]},

    { key: 'phones', label: 'شماره‌های تماس', type: 'array', maxItems: 3, children: [
        { key: 'number', label: 'شماره', type: 'tel', colSpan: 11,
          validators: [Validators.required, Validators.pattern(/^09\d{9}$/)] },
    ]},

    { key: 'bio', label: 'درباره من', type: 'textarea', validators: [Validators.maxLength(300)] },

    { key: 'terms', label: 'قوانین را می‌پذیرم', type: 'checkbox',
      validators: [Validators.requiredTrue],
      messages: { required: 'پذیرش قوانین الزامی است' } },
  ];

  save(value: Record<string, any>) {
    this.saving.set(true);
    this.api.register(value).subscribe({
      next: () => { this.saving.set(false); this.formCmp().form().markAsPristine(); },
      error: (err: HttpErrorResponse) => {
        this.saving.set(false);
        applyServerErrors(this.formCmp().form(), err.error?.errors);   // خطاهای Backend روی فیلدها
      },
    });
  }
}
```

```ts
// user.api.ts
@Injectable({ providedIn: 'root' })
export class UserApi {
  private http = inject(HttpClient);
  emailExists(email: string) {
    return this.http.get<boolean>('/api/users/email-exists', { params: { email } });
  }
  register(dto: unknown) { return this.http.post('/api/users', dto); }
}
```

**Type-safety:** خروجی `submitted` عمداً `Record<string, any>` است چون ساختار فرم در زمان اجرا از config می‌آید. در صفحه، یک بار با یک interface مشخص تبدیل کنید: `save(v: Record<string, any>) { const dto = v as RegisterDto; ... }`. Typed Forms کامل (با generic) فقط وقتی ممکن است که فرم را دستی بنویسید؛ این یک مصالحه آگاهانه برای انعطاف است.

## ۱۵. خطاهای Backend (.NET) روی فیلدها

ASP.NET در خطای اعتبارسنجی یک `ValidationProblemDetails` برمی‌گرداند. کلیدها PascalCase و آرایه‌ها به شکل `Phones[0].Number` هستند:

```csharp
// Controller / Minimal API
if (!ModelState.IsValid) return ValidationProblem(ModelState);
// یا با FluentValidation / قانون دامنه:
ModelState.AddModelError(nameof(dto.Email), "این ایمیل قبلاً ثبت شده است");
return ValidationProblem(ModelState);
```

خروجی JSON: `{ "errors": { "Email": ["..."], "Phones[0].Number": ["..."] } }`. این تابع آن را به مسیر Angular (`phones.0.number`) تبدیل می‌کند:

```ts
// utils/server-errors.ts
import { AbstractControl } from '@angular/forms';

export function applyServerErrors(form: AbstractControl, errors?: Record<string, string[]>) {
  Object.entries(errors ?? {}).forEach(([key, msgs]) => {
    const path = key
      .replace(/\[(\d+)\]/g, '.$1')
      .split('.').filter(Boolean)
      .map(p => (/^\d+$/.test(p) ? p : p[0].toLowerCase() + p.slice(1)))   // PascalCase → camelCase
      .join('.');

    const ctrl = (path && form.get(path)) || form;     // اگر فیلد پیدا نشد روی کل فرم
    ctrl.setErrors({ ...(ctrl.errors ?? {}), server: msgs[0] });
    ctrl.markAsTouched();
  });
}
```

خطای `server` با تغییر بعدی مقدار همان فیلد خودکار پاک می‌شود، چون validator ها دوباره اجرا می‌شوند.

## ۱۶. قابلیت‌های تکمیلی

### هشدار خروج بدون ذخیره

```ts
export interface HasForm { formCmp: () => AppFormComponent; }

export const unsavedChangesGuard: CanDeactivateFn<HasForm> = page =>
  !page.formCmp().form().dirty || confirm('تغییرات ذخیره‌نشده از بین می‌رود. خارج می‌شوید؟');
```

در route: `canDeactivate: [unsavedChangesGuard]`. صفحه باید `formCmp` را (مثل مثال بالا) داشته باشد.

### ذخیره خودکار پیش‌نویس (امن)

فیلدهای حساس را **هرگز** ذخیره نکنید:

```ts
import { debounceTime } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import isEqual from 'lodash-es/isEqual';        // npm i lodash-es

const SENSITIVE = ['password', 'confirmPassword', 'nationalCode'];

constructor() {
  // اگر خارج از constructor اجرا می‌کنید، DestroyRef را به takeUntilDestroyed بدهید
  this.formCmp().form().valueChanges.pipe(debounceTime(800), takeUntilDestroyed()).subscribe(v => {
    const safe = Object.fromEntries(Object.entries(v).filter(([k]) => !SENSITIVE.includes(k)));
    localStorage.setItem('draft', JSON.stringify(safe));
  });
}
```

### تغییر پویای validator

```ts
const ctrl = form.get('x')!;
ctrl.setValidators([Validators.required]);
ctrl.updateValueAndValidity();
```

## ۱۷. افزودن یک کنترل جدید (الگوی توسعه)

برای هر نوع ورودی جدید (مثلاً DatePicker شمسی) این سه کار کافی است:

1. کامپوننتی بسازید که از `BaseControl<T>` ارث می‌برد و هنگام تغییر مقدار `this.update(مقدار)` را صدا می‌زند.
2. نوع جدید را به `FieldType` اضافه کنید (مثلاً `'jalali-date'`).
3. یک `@case` در `field.component.ts` اضافه کنید.

```ts
// controls/jalali-date.component.ts (اسکلت)
@Component({
  selector: 'app-jalali-date', standalone: true, imports: [FieldErrorsComponent],
  template: `
    <div class="field" [class.has-error]="showError">
      <label [for]="id">{{ label() }}</label>
      <input [id]="id" [value]="display()" (change)="onPick($any($event.target).value)" (blur)="touch()" />
      <app-field-errors [control]="control" />
    </div>`,
})
export class JalaliDateComponent extends BaseControl<string> {
  label = input.required<string>();
  display = computed(() => this.toJalali(this.value()));     // نمایش شمسی
  onPick(text: string) { this.update(this.toIso(text)); }     // ذخیره میلادی/ISO
  // toJalali / toIso را با date-fns-jalali یا کتابخانه دلخواه پیاده کنید
}
```

مقدار را همیشه به‌صورت ISO/میلادی نگه دارید و فقط در UI شمسی نمایش دهید؛ سرور و دیتابیس ساده‌تر می‌مانند.

## ۱۸. تست‌های واحد

با Jasmine یا Jest (سینتکس یکسان است).

```ts
// validators/custom-validators.spec.ts
import { FormControl, FormGroup } from '@angular/forms';
import { nationalCode, passwordMatch } from './custom-validators';

describe('nationalCode', () => {
  const v = nationalCode();
  it('کد معتبر را می‌پذیرد', () => expect(v(new FormControl('0499370899'))).toBeNull());
  it('کد نامعتبر را رد می‌کند', () => expect(v(new FormControl('1234567890'))).toEqual({ nationalCode: true }));
  it('ارقام تکراری را رد می‌کند', () => expect(v(new FormControl('1111111111'))).toEqual({ nationalCode: true }));
  it('مقدار خالی را به required می‌سپارد', () => expect(v(new FormControl(''))).toBeNull());
});

describe('passwordMatch', () => {
  it('عدم تطابق را گزارش می‌کند', () => {
    const g = new FormGroup({ password: new FormControl('a1'), confirmPassword: new FormControl('b2') });
    expect(passwordMatch(g)).toEqual({ passwordMismatch: true });
  });
});
```

```ts
// services/form-factory.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { FormArray, FormGroup, Validators } from '@angular/forms';
import { FormFactory } from './form-factory.service';
import { FieldConfig } from '../models/field-config';

describe('FormFactory', () => {
  let factory: FormFactory;
  const fields: FieldConfig[] = [
    { key: 'name', label: 'نام', type: 'text', validators: [Validators.required] },
    { key: 'phones', label: 'تلفن‌ها', type: 'array', children: [{ key: 'number', label: 'شماره', type: 'tel' }] },
    { key: 'address', label: 'آدرس', type: 'group', children: [{ key: 'city', label: 'شهر', type: 'text' }] },
  ];

  beforeEach(() => { TestBed.configureTestingModule({}); factory = TestBed.inject(FormFactory); });

  it('فرم را از روی config می‌سازد', () => {
    const form = factory.build(fields);
    expect(form.get('name')).toBeTruthy();
    expect(form.get('address.city')).toBeTruthy();
    expect(form.get('phones') instanceof FormArray).toBeTrue();   // در Jest: toBe(true)
  });

  it('patch آیتم‌های آرایه را می‌سازد', () => {
    const form = factory.build(fields);
    factory.patch(form, fields, { name: 'Ali', phones: [{ number: '0912' }, { number: '0935' }] });
    expect((form.get('phones') as FormArray).length).toBe(2);
    expect(form.get('phones.1.number')!.value).toBe('0935');
    expect(form.pristine).toBeTrue();
  });

  it('reset آرایه را پاک می‌کند', () => {
    const form = factory.build(fields);
    (form.get('phones') as FormArray).push(factory.createArrayItem(fields[1]));
    factory.reset(form, fields, null);
    expect((form.get('phones') as FormArray).length).toBe(0);
  });

  it('required را اعمال می‌کند', () => {
    expect(factory.build(fields).get('name')!.valid).toBeFalse();
  });
});
```

برای تست کامپوننت `AppFormComponent`: با `TestBed.createComponent` و `fixture.componentRef.setInput('fields', [...])` فرم را بسازید، با `form().get('x')!.setValue(...)` مقدار بدهید و `submit()` را صدا بزنید؛ سپس خروجی `submitted` را بررسی کنید.

## ۱۹. نکات مهم و تله‌های رایج

1. **Reference ثابت برای `fields`**: اگر در هر change detection آرایه جدید بسازید، `form` دوباره ساخته می‌شود و ورودی کاربر پاک می‌شود. آن را property کلاس یا `signal` بگذارید.
2. **OnPush**: خطاها از `control.touched` می‌آیند که signal نیست. در این پیاده‌سازی کنترل‌ها را Default بگذارید. برای OnPush در Angular 18+ می‌توانید در `BaseControl` به `control.events` گوش دهید و یک signal به‌روز کنید.
3. **`form.value` در برابر `getRawValue()`**: فیلدهای disabled در `value` نمی‌آیند؛ با `includeDisabled` کنترل کنید.
4. **فیلد شرطی و validation**: فیلد مخفی disable می‌شود، پس validator آن اجرا نمی‌شود و در خروجی نمی‌آید. اگر هنگام پنهان شدن باید مقدارش پاک شود، در `applyVisibility` یک `ctrl.reset()` اضافه کنید.
5. **Async validator و `updateOn: 'blur'`**: برای فیلدهایی که API می‌زنند `blur` بگذارید تا با هر کلید درخواست نرود. `debounce` داخل validator هم لازم است.
6. **سرور هرگز کنار گذاشته نمی‌شود**: validation سمت Angular فقط UX است؛ validation اصلی باید سمت .NET باشد (FluentValidation یا Domain Rules در DDD).
7. **RTL**: همه‌جا Logical Properties (`margin-inline-start`, `padding-inline`). `left/right` ننویسید.
8. **Accessibility**: `label for`, `aria-invalid`, `role="alert"`، و `fieldset/legend` برای گروه‌ها و Radio در کد رعایت شده است.

## ۲۰. محدودیت‌های شناخته‌شده

این موارد عمداً ساده نگه داشته شده‌اند؛ برای پروژه واقعی یا Portfolio می‌توانید آن‌ها را توسعه دهید:

| محدودیت | راه توسعه |
| --- | --- |
| Typed Forms کامل ندارد (خروجی `Record<string, any>`) | تعریف config با generic و ساخت نوع از روی آن |
| `updateOn: 'submit'` پشتیبانی نمی‌شود | تنظیم `updateOn` روی کل `FormGroup` در `FormFactory.build` |
| کنترل‌های Multi-select، Upload، Autocomplete، DatePicker شمسی ندارد | الگوی بخش ۱۷ |
| فقط `visibleWhen`؛ `disabledWhen` و `requiredWhen` ندارد | افزودن به `FieldConfig` و `applyVisibility` |
| Wizard چندمرحله‌ای ندارد | یک `<app-wizard>` که چند `FieldConfig[]` (هر مرحله یک فرم) را پشت‌سرهم نشان دهد |
| تست E2E ندارد | Playwright یا Cypress روی صفحه ثبت‌نام |
| UI Library ندارد (HTML خام) | فقط template کنترل‌ها را با Angular Material یا PrimeNG عوض کنید؛ `BaseControl` و `FormFactory` ثابت می‌مانند |

## ۲۱. چک‌لیست قابلیت‌ها

| قابلیت | کجا |
| --- | --- |
| Reactive Forms + NonNullable | `FormFactory` |
| Validator های Sync / Async / Cross-field | بخش ۵ |
| ControlValueAccessor | `BaseControl` و کنترل‌ها |
| `touched / dirty / pending / disabled` | `BaseControl`، `submit()` |
| `updateOn: 'blur'` | `FieldConfig.updateOn` |
| FormGroup تو در تو، FormArray پویا (بازگشتی) | `app-field`، `FormFactory.patch` |
| فیلد شرطی در همه سطوح | `visibleWhen` |
| Edit mode و Reset کامل | `patch` و `reset` |
| `markAllAsTouched` + فوکوس اولین خطا | `submit()` |
| انتظار برای Async validator قبل از submit | `submit()` |
| پیام خطای مرکزی و قابل override | `VALIDATION_MESSAGES` |
| خطای سرور (.NET) | `applyServerErrors` |
| Layout گرید ۱۲ ستونی + Responsive + RTL | `colSpan` و استایل |
| Accessibility | کنترل‌ها |
| Unsaved changes / Autosave امن | بخش ۱۶ |
| تست واحد | بخش ۱۸ |

## ۲۲. مسیر پیشنهادی ادامه (برای Portfolio)

1. اجرای این کد در یک پروژه Angular تازه و رفع هر خطای نسخه‌ای احتمالی
2. افزودن DatePicker شمسی و Upload فایل
3. نوشتن `<app-wizard>` و تست‌های E2E
4. انتشار به‌صورت پکیج داخلی (Nx یا Angular library) و مستندسازی با Storybook
5. اتصال به یک Backend کوچک با .NET (Clean Architecture + FluentValidation) تا چرخه کامل خطای سرور نمایش داده شود
