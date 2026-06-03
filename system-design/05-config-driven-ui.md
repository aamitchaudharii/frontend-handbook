# 05 — Config-Driven UI

> **"When the structure of your UI is defined in data rather than code, non-engineers can change what the user sees without a deployment. That's a business capability, not just a technical pattern."**

Config-driven UI (also called data-driven UI or schema-driven UI) separates the **what** (configuration, schema) from the **how** (rendering logic). Instead of hardcoding form fields, table columns, dashboard layouts, or navigation structures in component code, you define them as data — JSON, TypeScript objects, or database records — and build a renderer that interprets that data at runtime. This document covers when this pattern adds value, how to design schemas that scale, how to build renderers, and where the approach breaks down.

---

## 📚 Table of Contents

1. [What Config-Driven UI Is](#1-what-config-driven-ui-is)
2. [When Config-Driven UI Adds Value](#2-when-config-driven-ui-adds-value)
3. [Form Schema Design](#3-form-schema-design)
4. [Building a Form Renderer](#4-building-a-form-renderer)
5. [Table/Grid Schema Design](#5-tablegrid-schema-design)
6. [Dashboard and Layout Config](#6-dashboard-and-layout-config)
7. [Navigation Config](#7-navigation-config)
8. [Validation in Config-Driven Forms](#8-validation-in-config-driven-forms)
9. [Dynamic Field Logic — Conditions and Dependencies](#9-dynamic-field-logic--conditions-and-dependencies)
10. [Config Stored in a Database](#10-config-stored-in-a-database)
11. [Versioning Config Schemas](#11-versioning-config-schemas)
12. [The Escape Hatch — Custom Components](#12-the-escape-hatch--custom-components)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What Config-Driven UI Is

In traditional development, a form looks like this:

```tsx
// Traditional: form structure hardcoded in JSX
function UserForm({ user, onSave }: UserFormProps) {
  return (
    <form>
      <Input name="firstName" label="First Name" required />
      <Input name="lastName" label="Last Name" required />
      <Input name="email" label="Email" type="email" required />
      <Select
        name="role"
        label="Role"
        options={[
          { label: "Admin", value: "admin" },
          { label: "User", value: "user" },
        ]}
      />
      <Button type="submit">Save</Button>
    </form>
  );
}
```

In config-driven development, the structure is data:

```typescript
// Config-driven: form structure as data
const userFormConfig: FormSchema = {
  fields: [
    { id: 'firstName', type: 'text',   label: 'First Name', required: true },
    { id: 'lastName',  type: 'text',   label: 'Last Name',  required: true },
    { id: 'email',     type: 'email',  label: 'Email',      required: true },
    { id: 'role',      type: 'select', label: 'Role',
      options: [{ label: 'Admin', value: 'admin' }, { label: 'User', value: 'user' }] },
  ],
};

// One renderer handles any form config
<FormRenderer schema={userFormConfig} onSubmit={handleSave} />
```

The same renderer renders any form — adding a new field to the user form is a data change, not a code change.

---

## 2. When Config-Driven UI Adds Value

### ✅ High-Value Use Cases

```
FORMS:
  Admin panels with many similar forms (user, product, category, etc.)
  CMS field configuration (content editors define their own fields)
  Survey builders (users create their own forms)
  Dynamic forms based on user type or feature flags

TABLES / GRIDS:
  Admin tables where columns change per user role
  Reporting tools where users configure their own views
  Data export tools with configurable column selection

DASHBOARDS:
  Business intelligence dashboards where layout is configurable
  Analytics widgets that non-engineers can rearrange
  Personalized landing pages

NAVIGATION:
  Menu structures that change per user role/feature flags
  CMS-managed navigation
  Multi-tenant apps where each tenant has different nav

RULE ENGINES:
  Pricing rules, discount logic
  Permission rules
  Notification routing
```

### ❌ When It's Overkill

```
ONE-OFF FORMS:
  A single login form with 2 fields — just write the JSX.
  The ceremony of defining a schema + renderer exceeds the benefit.

HIGHLY CUSTOM UI:
  If every form is completely different in appearance and behavior,
  config-driven adds abstraction without reducing code.

SIMPLE APPS:
  Small apps with < 5 forms/tables don't benefit enough.
  The schema design, renderer, and maintenance cost exceeds
  the saved development time.

COMPLEX INTERACTIONS:
  If a form has complex cross-field logic, animations, or
  completely custom layouts, forcing it into a generic schema
  creates a leaky abstraction.
```

---

## 3. Form Schema Design

### Basic Field Schema

```typescript
// Core field types
type FieldType =
  | "text"
  | "email"
  | "password"
  | "number"
  | "tel"
  | "url"
  | "textarea"
  | "select"
  | "multi-select"
  | "combobox"
  | "checkbox"
  | "radio"
  | "date"
  | "datetime"
  | "time"
  | "file"
  | "image"
  | "hidden"
  | "custom"; // escape hatch

interface FieldOption {
  label: string;
  value: string;
  disabled?: boolean;
  group?: string;
}

interface FieldSchema {
  // Required
  id: string;
  type: FieldType;
  label: string;

  // Common optional
  placeholder?: string;
  helpText?: string;
  defaultValue?: unknown;
  required?: boolean;
  disabled?: boolean;
  readOnly?: boolean;
  hidden?: boolean;

  // Select/radio/checkbox fields
  options?: FieldOption[] | string; // string = fetch URL

  // Text fields
  minLength?: number;
  maxLength?: number;
  pattern?: string;

  // Number fields
  min?: number;
  max?: number;
  step?: number;

  // File fields
  accept?: string;
  multiple?: boolean;

  // Layout
  width?: "full" | "half" | "third" | "quarter";
  group?: string; // visual grouping

  // Custom component (escape hatch)
  component?: string; // registered component name
  props?: Record<string, unknown>;

  // Dynamic behavior
  conditions?: FieldCondition[];
  dependsOn?: FieldDependency[];
}
```

### Form Schema

```typescript
interface FormSchema {
  id: string;
  title?: string;
  version: number; // for migration support

  // Fields
  fields: FieldSchema[];

  // Visual layout
  sections?: FormSection[];

  // Behavior
  submitLabel?: string;
  cancelLabel?: string;
  layout?: "vertical" | "horizontal" | "grid";

  // Validation
  validationMode?: "onChange" | "onBlur" | "onSubmit";
}

interface FormSection {
  id: string;
  title?: string;
  fields: string[]; // field IDs in this section
  collapsible?: boolean;
}
```

### Real-World Form Config Example

```typescript
const productFormSchema: FormSchema = {
  id: "product-form",
  title: "Product Details",
  version: 2,
  layout: "grid",

  sections: [
    {
      id: "basic",
      title: "Basic Info",
      fields: ["name", "sku", "description"],
    },
    {
      id: "pricing",
      title: "Pricing",
      fields: ["price", "comparePrice", "taxable"],
    },
    {
      id: "stock",
      title: "Inventory",
      fields: ["trackInventory", "quantity", "lowStockThreshold"],
    },
    { id: "media", title: "Media", fields: ["images", "featuredImage"] },
    {
      id: "meta",
      title: "SEO & Display",
      fields: ["metaTitle", "metaDescription", "status"],
    },
  ],

  fields: [
    {
      id: "name",
      type: "text",
      label: "Product Name",
      required: true,
      width: "full",
      maxLength: 200,
    },
    {
      id: "sku",
      type: "text",
      label: "SKU",
      width: "half",
      pattern: "^[A-Z0-9-]+$",
    },
    {
      id: "description",
      type: "textarea",
      label: "Description",
      width: "full",
      maxLength: 5000,
    },

    {
      id: "price",
      type: "number",
      label: "Price",
      required: true,
      min: 0,
      step: 0.01,
      width: "half",
    },
    {
      id: "comparePrice",
      type: "number",
      label: "Compare at Price",
      min: 0,
      step: 0.01,
      width: "half",
    },
    {
      id: "taxable",
      type: "checkbox",
      label: "Charge taxes on this product",
      defaultValue: true,
    },

    {
      id: "trackInventory",
      type: "checkbox",
      label: "Track inventory",
      defaultValue: false,
    },
    {
      id: "quantity",
      type: "number",
      label: "Available Quantity",
      min: 0,
      step: 1,
      width: "half",
      conditions: [{ field: "trackInventory", operator: "eq", value: true }],
    },
    {
      id: "lowStockThreshold",
      type: "number",
      label: "Low Stock Alert",
      min: 0,
      step: 1,
      width: "half",
      conditions: [{ field: "trackInventory", operator: "eq", value: true }],
    },

    {
      id: "images",
      type: "image",
      label: "Product Images",
      multiple: true,
      accept: "image/*",
    },
    {
      id: "featuredImage",
      type: "image",
      label: "Featured Image",
      accept: "image/*",
      width: "half",
    },

    {
      id: "metaTitle",
      type: "text",
      label: "SEO Title",
      maxLength: 60,
      width: "full",
    },
    {
      id: "metaDescription",
      type: "textarea",
      label: "SEO Description",
      maxLength: 160,
      width: "full",
    },
    {
      id: "status",
      type: "select",
      label: "Status",
      required: true,
      defaultValue: "draft",
      options: [
        { label: "Draft", value: "draft" },
        { label: "Active", value: "active" },
        { label: "Archived", value: "archived" },
      ],
    },
  ],
};
```

---

## 4. Building a Form Renderer

```tsx
// FormRenderer.tsx
import { useForm, Controller } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { buildValidationSchema } from "./buildValidationSchema";
import { evaluateConditions } from "./evaluateConditions";
import { FIELD_COMPONENTS } from "./fieldComponents";

interface FormRendererProps {
  schema: FormSchema;
  defaultValues?: Record<string, unknown>;
  onSubmit: (data: Record<string, unknown>) => void | Promise<void>;
  readOnly?: boolean;
}

export function FormRenderer({
  schema,
  defaultValues,
  onSubmit,
  readOnly,
}: FormRendererProps) {
  // Build Zod validation schema from field definitions
  const zodSchema = useMemo(
    () => buildValidationSchema(schema.fields),
    [schema],
  );

  const form = useForm({
    resolver: zodResolver(zodSchema),
    defaultValues: defaultValues ?? buildDefaults(schema.fields),
    mode: schema.validationMode ?? "onBlur",
  });

  const watchedValues = form.watch();

  // Render form sections or flat field list
  const sections = schema.sections
    ? schema.sections.map((section) => ({
        ...section,
        fields: section.fields
          .map((id) => schema.fields.find((f) => f.id === id)!)
          .filter(Boolean),
      }))
    : [{ id: "default", title: undefined, fields: schema.fields }];

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {sections.map((section) => (
        <FormSection key={section.id} title={section.title}>
          <div
            className={`form-grid form-grid--${schema.layout ?? "vertical"}`}
          >
            {section.fields.map((field) => (
              <FieldRenderer
                key={field.id}
                field={field}
                form={form}
                watchedValues={watchedValues}
                readOnly={readOnly}
              />
            ))}
          </div>
        </FormSection>
      ))}

      <div className="form-actions">
        <Button type="submit" loading={form.formState.isSubmitting}>
          {schema.submitLabel ?? "Save"}
        </Button>
      </div>
    </form>
  );
}

// FieldRenderer: renders a single field based on its type
function FieldRenderer({
  field,
  form,
  watchedValues,
  readOnly,
}: FieldRendererProps) {
  // Evaluate conditional visibility
  const isVisible =
    !field.conditions || evaluateConditions(field.conditions, watchedValues);

  if (!isVisible) return null;

  // Get the right component for this field type
  const FieldComponent = field.component
    ? FIELD_COMPONENTS[field.component] // custom registered component
    : FIELD_COMPONENTS[field.type]; // built-in type

  if (!FieldComponent) {
    console.warn(`No component registered for field type: ${field.type}`);
    return null;
  }

  const { errors } = form.formState;
  const error = errors[field.id];

  return (
    <div
      className={`form-field form-field--${field.width ?? "full"}`}
      aria-invalid={!!error}
    >
      <Controller
        name={field.id}
        control={form.control}
        render={({ field: controllerField, fieldState }) => (
          <FieldComponent
            {...controllerField}
            field={field}
            error={fieldState.error?.message}
            readOnly={readOnly || field.readOnly}
            disabled={field.disabled}
          />
        )}
      />
    </div>
  );
}
```

### Field Component Registry

```typescript
// fieldComponents.ts — register field type → React component
import { TextField } from "@/shared/components/TextField";
import { TextareaField } from "@/shared/components/TextareaField";
import { SelectField } from "@/shared/components/SelectField";
import { CheckboxField } from "@/shared/components/CheckboxField";
import { DateField } from "@/shared/components/DateField";
import { ImageUploadField } from "@/shared/components/ImageUploadField";

export const FIELD_COMPONENTS: Record<
  string,
  React.ComponentType<FieldComponentProps>
> = {
  text: TextField,
  email: TextField, // same component, type="email" passed via field.type
  password: TextField, // type="password"
  number: TextField, // type="number"
  textarea: TextareaField,
  select: SelectField,
  "multi-select": SelectField, // isMulti=true
  checkbox: CheckboxField,
  date: DateField,
  image: ImageUploadField,
};

// Register custom components at app startup
export function registerFieldComponent(
  name: string,
  component: React.ComponentType,
) {
  FIELD_COMPONENTS[name] = component;
}

// In app initialization:
registerFieldComponent("rich-text", RichTextEditor);
registerFieldComponent("color-picker", ColorPickerField);
registerFieldComponent("address-lookup", AddressLookupField);
```

---

## 5. Table/Grid Schema Design

```typescript
interface ColumnSchema<T = unknown> {
  id: string;
  field?: keyof T; // data property to display (can be nested: 'user.name')
  header: string;
  type?:
    | "text"
    | "number"
    | "date"
    | "boolean"
    | "badge"
    | "actions"
    | "custom";

  // Display
  width?: number | string;
  minWidth?: number;
  align?: "left" | "center" | "right";
  format?: "currency" | "percent" | "date" | "datetime" | "filesize";
  truncate?: boolean;

  // Behavior
  sortable?: boolean;
  filterable?: boolean;
  pinned?: "left" | "right";
  hidden?: boolean;
  resizable?: boolean;

  // Badge type: show colored chip
  badgeMap?: Record<string, { label: string; color: string }>;

  // Actions type
  actions?: ActionSchema[];

  // Custom renderer
  component?: string;
}

interface TableSchema<T = unknown> {
  id: string;
  columns: ColumnSchema<T>[];

  // Behavior
  rowKey: keyof T;
  selectable?: boolean;
  sortable?: boolean;
  filterable?: boolean;
  pageable?: boolean;
  defaultSort?: { field: string; direction: "asc" | "desc" };

  // Appearance
  density?: "compact" | "comfortable" | "spacious";
  striped?: boolean;
  stickyHeader?: boolean;
}
```

### Table Config Example

```typescript
const usersTableSchema: TableSchema<User> = {
  id: "users-table",
  rowKey: "id",
  selectable: true,
  sortable: true,
  pageable: true,
  stickyHeader: true,
  defaultSort: { field: "createdAt", direction: "desc" },

  columns: [
    {
      id: "name",
      field: "name",
      header: "User",
      type: "text",
      sortable: true,
      width: 200,
    },
    {
      id: "email",
      field: "email",
      header: "Email",
      type: "text",
      sortable: true,
      filterable: true,
    },
    {
      id: "role",
      field: "role",
      header: "Role",
      type: "badge",
      badgeMap: {
        admin: { label: "Admin", color: "purple" },
        user: { label: "User", color: "blue" },
        viewer: { label: "Viewer", color: "gray" },
      },
    },
    {
      id: "status",
      field: "isActive",
      header: "Status",
      type: "boolean",
      width: 80,
      component: "status-indicator",
    },
    {
      id: "createdAt",
      field: "createdAt",
      header: "Joined",
      type: "date",
      format: "date",
      sortable: true,
    },
    {
      id: "actions",
      type: "actions",
      header: "",
      width: 100,
      pinned: "right",
      actions: [
        { id: "edit", label: "Edit", icon: "pencil", onClick: "onEditUser" },
        {
          id: "delete",
          label: "Delete",
          icon: "trash",
          onClick: "onDeleteUser",
          variant: "danger",
        },
      ],
    },
  ],
};
```

---

## 6. Dashboard and Layout Config

```typescript
// Dashboard schema: grid of widgets
interface WidgetSchema {
  id: string;
  type: string; // registered widget type
  title?: string;
  props?: Record<string, unknown>;

  // Grid position (CSS Grid)
  grid: {
    col: number;
    row: number;
    colSpan: number;
    rowSpan: number;
  };
}

interface DashboardSchema {
  id: string;
  title?: string;
  layout: "grid" | "flexible";
  columns: number; // grid columns
  widgets: WidgetSchema[];
}

const analyticsDashboard: DashboardSchema = {
  id: "analytics",
  title: "Analytics Dashboard",
  layout: "grid",
  columns: 12,
  widgets: [
    {
      id: "revenue-kpi",
      type: "kpi-card",
      title: "Revenue",
      props: { metric: "total_revenue", period: "30d", format: "currency" },
      grid: { col: 1, row: 1, colSpan: 3, rowSpan: 1 },
    },
    {
      id: "orders-kpi",
      type: "kpi-card",
      title: "Orders",
      props: { metric: "order_count", period: "30d", format: "number" },
      grid: { col: 4, row: 1, colSpan: 3, rowSpan: 1 },
    },
    {
      id: "revenue-chart",
      type: "line-chart",
      title: "Revenue Over Time",
      props: { metric: "daily_revenue", period: "90d" },
      grid: { col: 1, row: 2, colSpan: 8, rowSpan: 2 },
    },
    {
      id: "top-products",
      type: "data-table",
      title: "Top Products",
      props: { report: "top_products", limit: 10 },
      grid: { col: 9, row: 2, colSpan: 4, rowSpan: 2 },
    },
  ],
};
```

---

## 7. Navigation Config

```typescript
interface NavItemSchema {
  id:       string;
  label:    string;
  icon?:    string;
  href?:    string;
  children?: NavItemSchema[];

  // Access control
  roles?:      string[];   // only shown to these roles
  permissions?: string[];  // only shown with these permissions
  feature?:    string;     // only shown when feature flag is on

  // Behavior
  external?:   boolean;    // open in new tab
  badge?:      string;     // dynamic badge source (e.g., 'unread_notifications')
}

// Role-specific navigation config
const adminNav: NavItemSchema[] = [
  { id: 'dashboard', label: 'Dashboard', icon: 'grid', href: '/admin' },
  {
    id: 'users', label: 'Users', icon: 'users',
    children: [
      { id: 'all-users',   label: 'All Users',   href: '/admin/users' },
      { id: 'invite',      label: 'Invite User', href: '/admin/users/invite' },
      { id: 'roles',       label: 'Roles',       href: '/admin/users/roles' },
    ],
  },
  { id: 'analytics', label: 'Analytics', icon: 'bar-chart', href: '/admin/analytics',
    permissions: ['view_analytics'] },
  { id: 'settings',  label: 'Settings',  icon: 'settings',  href: '/admin/settings' },
];

// Navigation renderer
function NavRenderer({ items }: { items: NavItemSchema[] }) {
  const { user } = useAuth();
  const { isEnabled } = useFeatureFlags();

  const visibleItems = items.filter(item => {
    if (item.roles?.length && !item.roles.includes(user.role)) return false;
    if (item.permissions?.length && !hasPermissions(user, item.permissions)) return false;
    if (item.feature && !isEnabled(item.feature)) return false;
    return true;
  });

  return (
    <nav>
      <ul>
        {visibleItems.map(item => (
          <NavItem key={item.id} item={item} />
        ))}
      </ul>
    </nav>
  );
}
```

---

## 8. Validation in Config-Driven Forms

```typescript
// Build Zod validation schema from field definitions
import { z } from "zod";

function buildValidationSchema(
  fields: FieldSchema[],
): z.ZodObject<z.ZodRawShape> {
  const shape: z.ZodRawShape = {};

  for (const field of fields) {
    let schema: z.ZodTypeAny = z.unknown();

    switch (field.type) {
      case "text":
      case "textarea": {
        let s = z.string();
        if (field.minLength)
          s = s.min(field.minLength, `Minimum ${field.minLength} characters`);
        if (field.maxLength)
          s = s.max(field.maxLength, `Maximum ${field.maxLength} characters`);
        if (field.pattern)
          s = s.regex(new RegExp(field.pattern), "Invalid format");
        schema = s;
        break;
      }
      case "email":
        schema = z.string().email("Invalid email address");
        break;
      case "number": {
        let s = z.number({ invalid_type_error: "Must be a number" });
        if (field.min !== undefined)
          s = s.min(field.min, `Minimum value is ${field.min}`);
        if (field.max !== undefined)
          s = s.max(field.max, `Maximum value is ${field.max}`);
        schema = s;
        break;
      }
      case "select":
        schema = z.string();
        break;
      case "checkbox":
        schema = z.boolean();
        break;
      case "date":
        schema = z.string().datetime({ message: "Invalid date" });
        break;
    }

    if (!field.required) {
      schema = schema.optional().nullable();
    } else {
      // Add required message
      if (schema instanceof z.ZodString) {
        schema = schema.min(1, `${field.label} is required`);
      }
    }

    shape[field.id] = schema;
  }

  return z.object(shape);
}
```

---

## 9. Dynamic Field Logic — Conditions and Dependencies

```typescript
interface FieldCondition {
  field: string; // ID of the field to check
  operator: "eq" | "ne" | "gt" | "lt" | "in" | "nin" | "empty" | "notEmpty";
  value?: unknown;
}

interface FieldConditions {
  operator?: "AND" | "OR"; // default AND
  conditions: FieldCondition[];
}

function evaluateConditions(
  conditions: FieldCondition[],
  values: Record<string, unknown>,
  logicalOp: "AND" | "OR" = "AND",
): boolean {
  const results = conditions.map((cond) => {
    const fieldValue = values[cond.field];

    switch (cond.operator) {
      case "eq":
        return fieldValue === cond.value;
      case "ne":
        return fieldValue !== cond.value;
      case "gt":
        return Number(fieldValue) > Number(cond.value);
      case "lt":
        return Number(fieldValue) < Number(cond.value);
      case "in":
        return Array.isArray(cond.value) && cond.value.includes(fieldValue);
      case "nin":
        return Array.isArray(cond.value) && !cond.value.includes(fieldValue);
      case "empty":
        return (
          !fieldValue || (Array.isArray(fieldValue) && fieldValue.length === 0)
        );
      case "notEmpty":
        return (
          !!fieldValue && (!Array.isArray(fieldValue) || fieldValue.length > 0)
        );
      default:
        return true;
    }
  });

  return logicalOp === "AND" ? results.every(Boolean) : results.some(Boolean);
}

// Usage in schema:
const shippingFormFields: FieldSchema[] = [
  {
    id: "country",
    type: "select",
    label: "Country",
    required: true,
    options: countryOptions,
  },

  {
    id: "state",
    type: "select",
    label: "State",
    required: true,
    options: usStates,
    // Only show for US addresses
    conditions: [{ field: "country", operator: "eq", value: "US" }],
  },

  {
    id: "province",
    type: "select",
    label: "Province",
    required: true,
    options: caProvinces,
    // Only show for Canadian addresses
    conditions: [{ field: "country", operator: "eq", value: "CA" }],
  },

  {
    id: "vatNumber",
    type: "text",
    label: "VAT Number",
    // Only show for EU business accounts
    conditions: [
      { field: "country", operator: "in", value: euCountryCodes },
      { field: "accountType", operator: "eq", value: "business" },
    ],
  },
];
```

---

## 10. Config Stored in a Database

When non-engineers need to modify configuration without deployments:

```typescript
// API: fetch form config from database
async function getFormConfig(formId: string): Promise<FormSchema> {
  const response = await fetch(`/api/admin/forms/${formId}/schema`);
  if (!response.ok) throw new Error(`Form ${formId} not found`);
  return response.json();
}

// Component: load config + render
function DynamicForm({ formId, onSubmit }: DynamicFormProps) {
  const { data: schema, isLoading, error } = useQuery({
    queryKey: ['form-schema', formId],
    queryFn:  () => getFormConfig(formId),
    staleTime: 5 * 60 * 1000, // cache for 5 minutes
  });

  if (isLoading) return <FormSkeleton />;
  if (error || !schema) return <FormError />;

  return <FormRenderer schema={schema} onSubmit={onSubmit} />;
}
```

### Schema Builder UI (for Admin)

```typescript
// An admin UI for editing form schemas
function FormSchemaBuilder({ formId }: { formId: string }) {
  const { data: schema } = useQuery(['form-schema', formId], ...);
  const updateSchema = useMutation(
    (updated: FormSchema) => api.put(`/admin/forms/${formId}/schema`, updated)
  );

  return (
    <DragDropContext onDragEnd={handleReorder}>
      <div className="schema-builder">
        <FieldList
          fields={schema?.fields ?? []}
          onAdd={addField}
          onEdit={editField}
          onRemove={removeField}
        />
        <FieldConfigPanel
          selectedField={selectedField}
          onChange={updateField}
        />
        <PreviewPanel schema={schema} />
      </div>
    </DragDropContext>
  );
}
```

---

## 11. Versioning Config Schemas

Schemas change over time. Version them to support migration.

```typescript
interface VersionedSchema {
  version: number;
  // ... schema content
}

const FORM_MIGRATIONS: Record<number, (schema: FormSchema) => FormSchema> = {
  // v1 → v2: rename 'type: text-area' to 'type: textarea'
  2: (schema) => ({
    ...schema,
    fields: schema.fields.map((field) => ({
      ...field,
      type: field.type === "text-area" ? "textarea" : field.type,
    })),
  }),

  // v2 → v3: convert options from string[] to FieldOption[]
  3: (schema) => ({
    ...schema,
    fields: schema.fields.map((field) => {
      if (field.type !== "select") return field;
      const options = Array.isArray(field.options)
        ? (field.options as string[]).map((opt) =>
            typeof opt === "string" ? { label: opt, value: opt } : opt,
          )
        : field.options;
      return { ...field, options };
    }),
  }),
};

const CURRENT_VERSION = 3;

function migrateSchema(schema: FormSchema): FormSchema {
  let current = schema;
  for (let v = (schema.version ?? 1) + 1; v <= CURRENT_VERSION; v++) {
    current = FORM_MIGRATIONS[v]?.(current) ?? current;
  }
  return { ...current, version: CURRENT_VERSION };
}
```

---

## 12. The Escape Hatch — Custom Components

No schema covers every case. Always provide an escape hatch.

```typescript
// Register custom components for special needs
registerFieldComponent('rich-text', ({ field, value, onChange, error }) => (
  <div className="rich-text-field">
    <label>{field.label}</label>
    <RichTextEditor
      value={value}
      onChange={onChange}
      toolbarOptions={field.props?.toolbarOptions}
    />
    {error && <span className="error">{error}</span>}
  </div>
));

registerFieldComponent('address-autocomplete', ({ field, value, onChange, error }) => (
  <GooglePlacesAutocomplete
    label={field.label}
    value={value}
    onSelect={(place) => onChange(place.formatted_address)}
    error={error}
  />
));
```

```typescript
// In schema: use custom component for a specific field
{
  id:        'headquarters',
  type:      'custom',
  label:     'Headquarters Address',
  component: 'address-autocomplete',
  required:  true,
  props:     { types: ['establishment'] },
}
```

---

## 13. Good Practices

### ✅ Type your schemas with TypeScript

```typescript
// Strong types prevent runtime errors from invalid config
const schema: FormSchema = { ... };
// TypeScript catches: missing required fields, wrong types, invalid enum values
```

### ✅ Validate schemas at load time

```typescript
// Validate config when it arrives (especially from database)
const schemaValidator = z.object({
  id: z.string(),
  version: z.number(),
  fields: z.array(fieldSchemaValidator),
});

function loadSchema(raw: unknown): FormSchema {
  const result = schemaValidator.safeParse(raw);
  if (!result.success) {
    throw new Error(`Invalid form schema: ${result.error.message}`);
  }
  return migrateSchema(result.data);
}
```

### ✅ Keep the renderer generic, keep logic in the schema

```typescript
// ✅ Schema encodes the intent
{ id: 'discount', type: 'number', conditions: [{ field: 'hasCoupon', operator: 'eq', value: true }] }

// ❌ Renderer encodes the intent (renderer knows too much)
// if (field.id === 'discount' && values.hasCoupon) { ... }
```

---

## 14. Bad Practices

### ❌ Putting imperative logic in schemas

```typescript
// ❌ Schema contains imperative code (not serializable to JSON, not configurable)
{
  id: 'price',
  type: 'number',
  validate: (value, formValues) => {
    // Arrow function inside schema — can't store in database!
    if (value < formValues.costPrice) return 'Price must exceed cost';
  }
}

// ✅ Schema is pure data (serializable), validation rules are declarative
{
  id: 'price',
  type: 'number',
  validationRules: [
    { rule: 'greaterThan', field: 'costPrice', message: 'Price must exceed cost' }
  ]
}
```

### ❌ Over-engineering the schema for every edge case

```typescript
// ❌ The schema becomes as complex as the code it replaces
// When your "config" requires a programming language to express,
// just write the component directly.

// If you need this level of complexity in your config:
{
  conditions: {
    OR: [
      {
        AND: [
          { field: "type", op: "eq", val: "A" },
          { field: "tier", op: "gt", val: 3 },
        ],
      },
      {
        AND: [
          { field: "type", op: "eq", val: "B" },
          { field: "active", op: "eq", val: true },
        ],
      },
    ];
  }
}
// → Write a custom component with the logic in code. It's clearer.
```

---

## 15. Common Mistakes

### Mistake 1 — Leaking renderer implementation into schemas

```typescript
// ❌ Schema knows too much about the renderer
{
  id: 'email',
  type: 'text',
  inputProps: { className: 'email-input', 'data-cy': 'email-field' } // renderer internals!
}

// ✅ Schema expresses intent, renderer handles presentation
{
  id:   'email',
  type: 'email',
  label: 'Email Address',
  // styling and test IDs are renderer concerns — not in schema
}
```

### Mistake 2 — No fallback for missing components

```typescript
// ❌ Renderer crashes if custom component isn't registered
const Component = FIELD_COMPONENTS[field.component]; // undefined!
return <Component />; // Error: undefined is not a function

// ✅ Graceful fallback
const Component = FIELD_COMPONENTS[field.component ?? field.type];
if (!Component) {
  return <UnknownFieldFallback field={field} />;
}
```

### Mistake 3 — No performance optimization for large schemas

```typescript
// ❌ Rebuilds Zod schema on every render
function FormRenderer({ schema }) {
  const zodSchema = buildValidationSchema(schema.fields); // expensive per render
  const form = useForm({ resolver: zodResolver(zodSchema) });
}

// ✅ Memoize schema derivation
function FormRenderer({ schema }) {
  const zodSchema = useMemo(
    () => buildValidationSchema(schema.fields),
    [schema], // only rebuild when schema changes
  );
  const form = useForm({ resolver: zodResolver(zodSchema) });
}
```

---

## 16. Interview-Level Explanation

> **"What is config-driven UI? When would you use it and how do you design the schema?"**

**Strong answer:**

> "Config-driven UI separates the structure of a UI component from its rendering logic. Instead of hardcoding a form's fields in JSX, you define them as a data schema — a TypeScript object or JSON — and build a renderer that interprets that schema. The renderer is written once; adding new forms or tables is a data change, not a code change.
>
> The highest-value use cases are admin panels with many similar forms, CMS platforms where content editors configure their own fields, multi-tenant applications where each tenant has a different UI configuration, and any tool where non-engineers need to modify what users see without requiring a deployment.
>
> Schema design is the critical part. The schema must be serializable (no functions inside JSON), declarative rather than imperative, and versioned to support migration as requirements evolve. Field schemas should express intent — 'this field is required,' 'this field appears when condition X is true' — not renderer mechanics like CSS class names.
>
> The hardest part is dynamic behavior: fields that appear or disappear based on other fields' values, validations that cross-reference other fields, options that load from an API. I model these with declarative condition objects — `{ field: 'country', operator: 'eq', value: 'US' }` — that the renderer evaluates by watching form values. This keeps the schema serializable while expressing meaningful behavior.
>
> Critically, you always need an escape hatch. No schema covers every case, and forcing a complex one-off into a generic schema creates a worse abstraction than just writing a custom component. I maintain a component registry where custom components can be registered by name and referenced from the schema. When the generic renderer can't express the behavior, you register a custom component that can.
>
> The failure mode is over-engineering: building a config schema so expressive that it's essentially a programming language, at which point you've invented a worse programming language than the one you already have."

---

## 17. Exercises

### Exercise 1 — Design a schema

Design a complete FormSchema for a job application form that includes:

- Personal info: name, email, phone, location
- Work experience: current title, current company, years of experience, employment type (dropdown)
- A textarea for cover letter
- A file upload for resume
- A "how did you hear about us" select
- A checkbox: "I agree to the terms"
- Conditional: if employment type is "freelance", show a "portfolio URL" field

<details>
<summary>Solution</summary>

```typescript
const jobApplicationSchema: FormSchema = {
  id: "job-application",
  title: "Apply for Position",
  version: 1,
  layout: "grid",

  sections: [
    {
      id: "personal",
      title: "Personal Information",
      fields: ["firstName", "lastName", "email", "phone", "location"],
    },
    {
      id: "experience",
      title: "Work Experience",
      fields: [
        "currentTitle",
        "currentCompany",
        "yearsExp",
        "employmentType",
        "portfolioUrl",
      ],
    },
    {
      id: "application",
      title: "Your Application",
      fields: ["coverLetter", "resume", "heardAbout"],
    },
    { id: "terms", title: "", fields: ["agreeTerms"] },
  ],

  fields: [
    {
      id: "firstName",
      type: "text",
      label: "First Name",
      required: true,
      width: "half",
    },
    {
      id: "lastName",
      type: "text",
      label: "Last Name",
      required: true,
      width: "half",
    },
    {
      id: "email",
      type: "email",
      label: "Email Address",
      required: true,
      width: "half",
    },
    { id: "phone", type: "tel", label: "Phone Number", width: "half" },
    {
      id: "location",
      type: "text",
      label: "City, Country",
      required: true,
      width: "full",
    },

    {
      id: "currentTitle",
      type: "text",
      label: "Current Job Title",
      required: true,
      width: "half",
    },
    {
      id: "currentCompany",
      type: "text",
      label: "Current Company",
      width: "half",
    },
    {
      id: "yearsExp",
      type: "number",
      label: "Years of Experience",
      required: true,
      min: 0,
      max: 50,
      step: 1,
      width: "half",
    },
    {
      id: "employmentType",
      type: "select",
      label: "Employment Type",
      required: true,
      width: "half",
      options: [
        { label: "Full-time", value: "full-time" },
        { label: "Part-time", value: "part-time" },
        { label: "Freelance", value: "freelance" },
        { label: "Contract", value: "contract" },
      ],
    },
    {
      id: "portfolioUrl",
      type: "url",
      label: "Portfolio URL",
      width: "full",
      helpText: "Required for freelance applicants",
      conditions: [
        { field: "employmentType", operator: "eq", value: "freelance" },
      ],
      required: false,
    }, // required enforced conditionally via custom validator

    {
      id: "coverLetter",
      type: "textarea",
      label: "Cover Letter",
      required: true,
      minLength: 100,
      maxLength: 2000,
      width: "full",
      helpText: "Tell us why you're the right fit (100–2000 characters)",
    },
    {
      id: "resume",
      type: "file",
      label: "Resume / CV",
      required: true,
      accept: ".pdf,.doc,.docx",
      width: "half",
    },
    {
      id: "heardAbout",
      type: "select",
      label: "How did you hear about us?",
      width: "half",
      options: [
        { label: "LinkedIn", value: "linkedin" },
        { label: "Job Board", value: "job-board" },
        { label: "Referral", value: "referral" },
        { label: "Company Site", value: "website" },
        { label: "Other", value: "other" },
      ],
    },

    {
      id: "agreeTerms",
      type: "checkbox",
      label: "I agree to the Terms of Service and Privacy Policy",
      required: true,
      width: "full",
    },
  ],
};
```

</details>

---

## 🔗 Related Topics

- [`system-design/01-large-scale-architecture.md`](./01-large-scale-architecture.md) — Architectural context for config patterns
- [`system-design/07-plugin-systems.md`](./07-plugin-systems.md) — Plugin patterns (similar extensibility approach)
- [`system-design/04-state-management-design.md`](./04-state-management-design.md) — Form state management
- [`patterns/02-strategy.md`](../patterns/02-strategy.md) — Strategy pattern underlies field type rendering

---

<div align="center">

**Next:** [`system-design/06-event-driven-frontend.md`](./06-event-driven-frontend.md) →

</div>
