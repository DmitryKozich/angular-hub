# Creating a Custom Form Control with ControlValueAccessor

In frontend development, we frequently deal with forms. Angular offers powerful tools for working with them through `FormsModule` and `ReactiveFormsModule`. These modules allow us to easily build forms, manage their state, and apply validation.

However, in real-world applications, we often need to create custom, reusable controls whose behavior goes beyond standard `<input>`, `<select>`, or `<textarea>` elements. For example, we might need a dropdown with async data loading, an autocomplete, a date picker, or even a complex component with nested fields.

For such cases, Angular provides the `ControlValueAccessor` interface, which allows us to build our own form controls and integrate them into both reactive and template-driven forms just like any built-in form element.

Let’s go through a practical example.

---

## Creating a Form with Standard Controls

Suppose we have a user entity that can be represented by the following interface:

```ts
export interface User {
  name: string;
  username: string;
  email: string;
  address: Address;
  phone: string;
}

export interface Address {
  street: string;
  city: string;
  zipcode: string;
}
```

We need a form that allows creating such users. The component might look like this:

```ts
type UserForm = { [K in keyof User]: AbstractControl };

@Component({
  ...
  imports: [ReactiveFormsModule],
})
export class UserFormComponent {
  protected userForm = new FormGroup<UserForm>({
    name: new FormControl(''),
    email: new FormControl(''),
    username: new FormControl(''),
    address: new FormGroup({
      street: new FormControl(''),
      city: new FormControl(''),
      zipcode: new FormControl(''),
    }),
    phone: new FormControl(''),
  });

  protected get addressForm(): FormGroup {
    return this.userForm.controls.address as FormGroup;
  }

  ...
}
```

And the HTML template might look like this:

```html
<form [formGroup]="userForm" class="user-form">
  <div class="form-group">
    <label for="name">Name</label>
    <input type="text" class="form-control" id="name" formControlName="name" />
  </div>

  <div class="form-group">
    <label for="email">Email</label>
    <input type="email" class="form-control" id="email" formControlName="email" />
  </div>

  <div class="form-group">
    <label for="username">Username</label>
    <input type="text" class="form-control" id="username" formControlName="username" />
  </div>

  <div class="form-group">
    <label for="phone">Phone</label>
    <input type="text" class="form-control" id="phone" formControlName="phone" />
  </div>

  <ng-container [formGroup]="addressForm">
    <div class="form-group">
      <label for="street">Street</label>
      <input type="text" class="form-control" id="street" formControlName="street" />
    </div>

    <div class="form-group">
      <label for="city">City</label>
      <input type="text" class="form-control" id="city" formControlName="city" />
    </div>

    <div class="form-group">
      <label for="zipcode">Zipcode</label>
      <input type="text" class="form-control" id="zipcode" formControlName="zipcode" />
    </div>
  </ng-container>
</form>
```

This works well, but here’s the problem: what if we also need an address form elsewhere, such as in a shipping form? Duplicating the form logic and layout would lead to maintainability issues. Instead, we can create a custom form control that can be embedded using `ngModel`, `formControl`, or `formControlName`. That’s where `ControlValueAccessor` comes in.

---

## What is ControlValueAccessor?

This is an interface that lets Angular interact with your custom component just like it does with native form controls. It provides support for two-way data binding and participates in the form’s lifecycle. The interface includes:

```ts
interface ControlValueAccessor {
  writeValue(obj: any): void;
  registerOnChange(fn: any): void;
  registerOnTouched(fn: any): void;
  optional setDisabledState(isDisabled: boolean): void;
}
```

---

## Creating a Custom Control

We begin by extracting the address form into a separate component:

```ts
type AddressForm = { [K in keyof Address]: AbstractControl };

@Component({
  selector: 'app-address-control',
  imports: [ReactiveFormsModule],
  ...
})
export class AddressControlComponent {
  protected addressFrom = new FormGroup<AddressForm>({
    street: new FormControl(''),
    city: new FormControl(''),
    zipcode: new FormControl(''),
  });
}
```

```html
<ng-container [formGroup]="addressFrom">
  <div class="form-group">
    <label for="street">Street</label>
    <input type="text" class="form-control" id="street" formControlName="street" />
  </div>

  <div class="form-group">
    <label for="city">City</label>
    <input type="text" class="form-control" id="city" formControlName="city" />
  </div>

  <div class="form-group">
    <label for="zipcode">Zipcode</label>
    <input type="text" class="form-control" id="zipcode" formControlName="zipcode" />
  </div>
</ng-container>
```

Next, we implement `ControlValueAccessor` in the `AddressControlComponent` class. A good practice is to create an abstract base class that implements the interface so multiple custom controls in the app can extend it:

```ts
export abstract class CustomControl implements ControlValueAccessor {
  protected onChange: any = () => {};
  protected onTouched: any = () => {};

  public abstract writeValue(obj: any): void;

  public registerOnChange(fn: any): void {
    this.onChange = fn;
  }

  public registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }

  public abstract setDisabledState?(isDisabled: boolean): void;
}
```

Then we implement it in the component:

```ts
type AddressForm = { [K in keyof Address]: AbstractControl };

@Component({
  selector: 'app-address-control',
  imports: [ReactiveFormsModule],
  ...
})
export class AddressControlComponent extends CustomControl implements OnInit, OnDestroy {
  protected addressFrom = new FormGroup<AddressForm>({
    street: new FormControl(''),
    city: new FormControl(''),
    zipcode: new FormControl(''),
  });

  private subscription = Subscription.EMPTY;

  ngOnInit(): void {
    this.subscription = this.addressFrom.valueChanges.subscribe((value) => {
      this.onChange(value);
    });
  }

  public override writeValue(address: Address): void {
    if (address) {
      this.addressFrom.patchValue(address, { emitEvent: false });
    }
  }

  public override setDisabledState(isDisabled: boolean): void {
    isDisabled ? this.addressFrom.disable() : this.addressFrom.enable();
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

What’s happening here:

1. Angular calls `registerOnChange` during form initialization and provides a callback for value changes. We store this callback locally.
2. It also calls `registerOnTouched` with a callback to mark the control as touched.
3. The `writeValue` method is triggered when the parent form writes a new value into the control.
4. `setDisabledState` is optional, and allows the parent to enable or disable the control.

But this is not enough — the component still won’t work as a form control. To make Angular aware of it, we must register it under the `NG_VALUE_ACCESSOR` provider:

```ts
@Component({
  selector: 'app-address-control',
  imports: [ReactiveFormsModule],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      multi: true,
      useExisting: AddressControlComponent,
    },
  ],
  ...
})
export class AddressControlComponent extends CustomControl implements OnInit, OnDestroy {
  ...
}
```

Now, directives like `ngModel`, `formControl`, and `formControlName` will recognize the component and treat it as a proper form control.

---

## Integrating the Custom Control into a Form

Let’s see how the `UserFormComponent` looks with the custom control:

```ts
type UserForm = { [K in keyof User]: AbstractControl };

@Component({
  imports: [ReactiveFormsModule, AddressControlComponent],
  ...
})
export class UserFormComponent {
  protected userForm = new FormGroup<UserForm>({
    name: new FormControl(''),
    email: new FormControl(''),
    username: new FormControl(''),
    // Now instead of FormGroup for the address, FormControl will be used
    address: new FormControl(),
    phone: new FormControl(''),
  });

  ...
}
```

```html
<form [formGroup]="userForm" class="user-form">
  <div class="form-group">
    <label for="name">Name</label>
    <input type="text" class="form-control" id="name" formControlName="name" />
  </div>

  <div class="form-group">
    <label for="email">Email</label>
    <input type="email" class="form-control" id="email" formControlName="email" />
  </div>

  <div class="form-group">
    <label for="username">Username</label>
    <input type="text" class="form-control" id="username" formControlName="username" />
  </div>

  <div class="form-group">
    <label for="phone">Phone</label>
    <input type="text" class="form-control" id="phone" formControlName="phone" />
  </div>

  <app-address-control formControlName="address"></app-address-control>
</form>
```

---

## Validation

Once we have a working custom control, we might want to build in validation, so we don’t have to redefine it for every form. To do this, we implement the `Validator` interface in the base class:

```ts
export abstract class CustomControl implements ControlValueAccessor, Validator {
  protected onChange: any = () => {};
  protected onTouched: any = () => {};
  protected onValidatorChange: any = () => {};

  public abstract writeValue(obj: any): void;

  public registerOnChange(fn: any): void {
    this.onChange = fn;
  }

  public registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }

  public abstract setDisabledState?(isDisabled: boolean): void;

  registerOnValidatorChange(fn: () => void): void {
    this.onValidatorChange = fn;
  }

  public abstract validate(control: AbstractControl): ValidationErrors | null;
}
```

This interface has two methods:

1. `validate`: used to validate the current control value. It runs when the control value changes.
2. `registerOnValidatorChange`: registers a callback that allows us to manually trigger validation. This is useful when validation depends on inputs other than the control’s value.

Here’s how we implement it in the component:

```ts
type AddressForm = { [K in keyof Address]: AbstractControl };

@Component({
  ...
  imports: [ReactiveFormsModule],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      multi: true,
      useExisting: AddressControlComponent,
    },
    {
      provide: NG_VALIDATORS,
      multi: true,
      useExisting: AddressControlComponent,
    },
  ],
})
export class AddressControlComponent extends CustomControl implements OnInit, OnDestroy {
  protected addressFrom = new FormGroup<AddressForm>({
    street: new FormControl('', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(100),
    ]),
    city: new FormControl('', [
      Validators.required,
      Validators.pattern(/^[a-zA-Z\s\-]+$/),
      Validators.minLength(2),
    ]),
    zipcode: new FormControl('', [
      Validators.required,
      Validators.pattern(/^\d{5}$/),
    ]),
  });

  ...

  public override validate(control: AbstractControl): ValidationErrors | null {
    return this.addressFrom.valid
      ? null
      : { invalidAddress: true, ...this.collectControlErrors() };
  }

  private collectControlErrors(): ValidationErrors {
    const fields = Object.keys(this.addressFrom.controls) as (keyof Address)[];
    return fields.reduce<ValidationErrors>((errors, field) => {
      const errorsField = this.addressFrom.get(field)?.errors;
      if (errorsField) {
        errors[field] = errorsField;
      }
      return errors;
    }, {});
  }

  ...
}
```

1. We define validators for the `addressFrom` controls.
2. We implement the `validate` method to return `null` if the form is valid, or an error object if it’s not.
3. We register the component under `NG_VALIDATORS` so Angular knows it provides validation logic.

The component’s template and usage remain the same.

---

## Conclusion

`ControlValueAccessor` allows you to create fully functional custom controls that integrate seamlessly with Angular forms. By implementing the required methods and registering the component under `NG_VALUE_ACCESSOR`, you enable support for value changes, validation, touched/dirty states, and disabling. This is an essential tool for building reusable and non-standard form controls in Angular.
