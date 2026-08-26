# Câu hỏi & Trả lời Phỏng vấn: Tailwind CSS, Ant Design, shadcn/ui

## 📘 TAILWIND CSS

### 1. Tailwind CSS là gì? Sự khác biệt với CSS truyền thống?
**Trả lời:**
- Tailwind là một **utility-first CSS framework** - cung cấp những class nhỏ, có mục đích cụ thể
- Thay vì viết CSS toàn bộ, bạn combine các utility class: `<div class="flex gap-4 bg-blue-500 p-4">`
- **Ưu điểm:**
  - Nhanh hơn (không cần viết CSS riêng)
  - Consistent design system
  - Dễ maintain (styles gắn liền với HTML)
  - Bundle size tối ưu (PurgeCSS loại bỏ unused styles)
- **Nhược điểm:**
  - HTML dài, khó đọc
  - Learning curve
  - Không flexible như CSS custom

### 2. Tailwind CSS hoạt động như thế nào?
**Trả lời:**
- Tailwind scan tất cả files (`.jsx`, `.html`, etc) để tìm class names
- Parse các class names và generate CSS tương ứng
- Output một single CSS file với tất cả styles cần thiết
- **Build process:**
  ```bash
  Input: <div class="flex p-4 bg-blue-500">
  Output: CSS file chứa .flex, .p-4, .bg-blue-500
  ```
- Sử dụng PurgeCSS để remove unused styles

### 3. Làm sao tạo custom style trong Tailwind?
**Trả lời:**
```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        'brand-blue': '#1e40af',
      },
      spacing: {
        '128': '32rem',
      },
      borderRadius: {
        '4xl': '2rem',
      }
    }
  }
}

// Sử dụng:
<div class="bg-brand-blue p-128 rounded-4xl">
```

### 4. Làm sao quản lý Tailwind CSS cho project lớn?
**Trả lời:**
- **Component extraction:** Tách các group class thành component
  ```jsx
  function Button({ children }) {
    return <button className="px-4 py-2 bg-blue-500 rounded hover:bg-blue-600">
      {children}
    </button>
  }
  ```
- **@apply directive:** Tạo reusable styles
  ```css
  @layer components {
    .btn-primary {
      @apply px-4 py-2 bg-blue-500 rounded hover:bg-blue-600;
    }
  }
  ```
- **Utility composition:** Tạo custom utilities
- **CSS-in-JS:** Combine với styled-components nếu cần

### 5. Tailwind có thể responsive không? Làm sao?
**Trả lời:**
- Tailwind có breakpoint system tích hợp
- Sử dụng breakpoint prefix: `sm:`, `md:`, `lg:`, `xl:`, `2xl:`
  ```jsx
  <div className="w-full md:w-1/2 lg:w-1/3">
    {/* Full width mobile, 50% tablet, 33% desktop */}
  </div>
  ```
- Có thể customize breakpoints:
  ```javascript
  theme: {
    screens: {
      'mobile': '320px',
      'tablet': '768px',
      'desktop': '1024px',
    }
  }
  ```

### 6. Làm sao optimize Tailwind CSS bundle size?
**Trả lời:**
- **Content paths:** Cấu hình đúng file cần scan
  ```javascript
  content: ["./src/**/*.{js,jsx,ts,tsx}"]
  ```
- **PurgeCSS:** Tự động loại bỏ unused styles
- **Avoid dynamic class names:** ❌ `className={`p-${value}`}` 
  ✅ `className={value === 4 ? 'p-4' : 'p-8'}`
- **Tree-shaking:** Loại bỏ plugins không dùng
- **Production build:** Minify CSS

---

## 🐜 ANT DESIGN (antd)

### 1. Ant Design là gì? Khi nào dùng?
**Trả lời:**
- Một **enterprise-class component library** cho React
- Cung cấp 50+ production-ready components
- Tuân theo **Ant Design specification** (design system của Alibaba)
- **Khi nào dùng:**
  - Admin dashboard, enterprise apps
  - Cần nhiều form, table, modal phức tạp
  - Khi time-to-market là ưu tiên
  - Team không có QA tuyệt vời (antd components đã tested)

### 2. Làm sao setup Ant Design?
**Trả lời:**
```bash
# Install
npm install antd

# Import
import { Button, Input, Form, Table } from 'antd';
import 'antd/dist/reset.css'; // v5+
```
- **Optional:** Customize theme
  ```javascript
  import { ConfigProvider } from 'antd';
  
  <ConfigProvider theme={{ token: { colorPrimary: '#1890ff' } }}>
    <App />
  </ConfigProvider>
  ```

### 3. Form handling trong Ant Design?
**Trả lời:**
```jsx
import { Form, Input, Button, message } from 'antd';

function MyForm() {
  const [form] = Form.useForm();
  
  const onFinish = (values) => {
    console.log('Success:', values);
  };
  
  return (
    <Form form={form} onFinish={onFinish} layout="vertical">
      <Form.Item
        label="Email"
        name="email"
        rules={[
          { required: true, message: 'Hãy nhập email!' },
          { type: 'email', message: 'Email không hợp lệ!' }
        ]}
      >
        <Input />
      </Form.Item>
      
      <Form.Item
        label="Password"
        name="password"
        rules={[{ required: true }]}
      >
        <Input.Password />
      </Form.Item>
      
      <Form.Item>
        <Button type="primary" htmlType="submit">
          Submit
        </Button>
      </Form.Item>
    </Form>
  );
}
```

### 4. Làm sao display data với Table?
**Trả lời:**
```jsx
import { Table } from 'antd';

function DataTable() {
  const columns = [
    {
      title: 'Name',
      dataIndex: 'name',
      key: 'name',
      sorter: (a, b) => a.name.localeCompare(b.name),
    },
    {
      title: 'Age',
      dataIndex: 'age',
      key: 'age',
    },
    {
      title: 'Action',
      key: 'action',
      render: (_, record) => (
        <span>
          <a>Edit {record.name}</a>
        </span>
      ),
    },
  ];

  const data = [
    { key: '1', name: 'John Brown', age: 32 },
    { key: '2', name: 'Jim Green', age: 42 },
  ];

  return (
    <Table 
      columns={columns} 
      dataSource={data}
      pagination={{ pageSize: 50 }}
      loading={false}
    />
  );
}
```

### 5. Ant Design theme customization?
**Trả lời:**
```javascript
// Cách 1: ConfigProvider (v5+)
<ConfigProvider
  theme={{
    token: {
      colorPrimary: '#1890ff',
      borderRadius: 6,
      fontFamily: 'Arial',
    },
    components: {
      Button: {
        borderRadius: 8,
      },
    },
  }}
>
  <App />
</ConfigProvider>

// Cách 2: CSS variables (v5+)
// theme={{ token: { ... } }}
```

### 6. Ưu & nhược điểm Ant Design?
**Trả lời:**
**Ưu:**
- Comprehensive component library
- Enterprise-ready (well-tested)
- Good documentation
- Consistent design

**Nhược:**
- Bundle size lớn (~1.5MB)
- Styling khó customize (limited vs Tailwind)
- Heavyweight cho small projects
- Opinionated (khó thay đổi design)

---

## ✨ SHADCN/UI

### 1. shadcn/ui là gì? Khác Ant Design như thế nào?
**Trả lời:**
- **shadcn/ui** không phải là package library mà là **copy-paste component collection**
- Build trên top của **Radix UI** (headless components) + **Tailwind CSS**
- Bạn copy component code vào project, không import từ package
- **Khác Ant Design:**
  - Lightweight, flexible hơn
  - Dễ customize (vì code ở local)
  - Build size tối ưu (chỉ dùng gì copy đó)
  - Tailwind-first (không CSS file nặng)

### 2. Làm sao setup shadcn/ui?
**Trả lời:**
```bash
# Init
npx shadcn-ui@latest init

# Copy component
npx shadcn-ui@latest add button
npx shadcn-ui@latest add form
npx shadcn-ui@latest add table
```
- Sau đó component tự động generate trong `components/ui/`
- Import và dùng bình thường:
  ```jsx
  import { Button } from "@/components/ui/button"
  ```

### 3. Form handling với shadcn/ui?
**Trả lời:**
```jsx
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Form, FormField, FormItem, FormLabel, FormControl } from "@/components/ui/form"
import { useForm } from "react-hook-form"
import { z } from "zod"

const formSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export default function MyForm() {
  const form = useForm({
    resolver: zodResolver(formSchema),
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input {...field} />
              </FormControl>
            </FormItem>
          )}
        />
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  )
}
```

### 4. Làm sao customize shadcn/ui component?
**Trả lời:**
- Vì code ở local, bạn có thể tự do chỉnh sửa
  ```jsx
  // components/ui/button.tsx
  export const Button = React.forwardRef(({ 
    className, 
    ...props 
  }, ref) => (
    <button
      className={cn(
        "px-4 py-2 bg-blue-500 text-white rounded",
        className
      )}
      ref={ref}
      {...props}
    />
  ))
  ```
- Điều này khác Ant Design rất nhiều (Ant Design phải override theme)

### 5. Khi nào dùng shadcn/ui?
**Trả lời:**
- ✅ Nextjs projects
- ✅ Tailwind-first projects
- ✅ Cần flexibility cao
- ✅ Small-medium projects
- ✅ Khi cần minimize bundle size
- ❌ Không nên: CMS lớn, enterprise apps cần UI consistency

### 6. shadcn/ui có mất thời gian copy-paste không?
**Trả lời:**
- **Ngắn hạn:** Đúng, hơi lâu hơn `npm install antd`
- **Dài hạn:** Đáng giá vì:
  - Customization dễ hơn nhiều
  - Bundle size nhỏ
  - Không phụ thuộc library updates
  - Code control hoàn toàn

---

## 🔄 SO SÁNH 3 FRAMEWORK

| Tiêu chí | Tailwind | Ant Design | shadcn/ui |
|---------|----------|-----------|-----------|
| **Loại** | Utility CSS | Component Library | Copy-paste Components |
| **Bundle Size** | Nhỏ (~15KB) | Lớn (~1.5MB) | Nhỏ (~50KB) |
| **Setup Time** | Nhanh | Nhanh | Trung bình |
| **Customization** | Rất dễ | Khó | Rất dễ |
| **Component Library** | Không có | 50+ sẵn | 30+ sẵn |
| **Learning Curve** | Trung bình | Dễ | Trung bình |
| **Best For** | Bất kỳ project | Enterprise apps | Nextjs/Tailwind projects |
| **UI Consistency** | Phụ thuộc bạn | Cao | Cao (tuỳ chỉnh) |

---

## 💡 CÂUHỎI THỰC TẾ TRONG PHỎNG VẤN

### 1. Bạn sẽ chọn framework nào cho admin dashboard?
**Trả lời:**
"Mình sẽ chọn **Ant Design** vì:
- Có sẵn Form, Table, Modal phức tạp (tiết kiệm thời gian)
- Enterprise-ready components
- Admin dashboard cần consistency cao
- Nếu client không yêu cầu custom design, Ant Design rất phù hợp"

### 2. Làm sao chia sẻ button component giữa các team?
**Trả lời:**
```jsx
// Cách 1: Component library (Tailwind)
export const Button = ({ variant, size, ...props }) => (
  <button className={cn(baseStyles, variants[variant], sizes[size])} {...props} />
)

// Cách 2: Publish npm package (với shadcn/ui)
// Tạo npm package chứa các component tái sử dụng

// Cách 3: Design System (Ant Design)
<ConfigProvider theme={sharedTheme}>
  <App />
</ConfigProvider>
```

### 3. Performance issue với Ant Design, làm sao tối ưu?
**Trả lời:**
```jsx
// 1. Code splitting
const Table = lazy(() => import('antd/lib/table'));

// 2. Chỉ import cần thiết
import Button from 'antd/lib/button'; // Thay vì import { Button } from 'antd'

// 3. ConfigProvider tại root
<ConfigProvider theme={{ ... }}>
  <App />
</ConfigProvider>

// 4. Virtual scroll cho Table lớn
<Table virtual {...props} />
```

### 4. Chuyển từ Ant Design sang Tailwind, strategy là gì?
**Trả lời:**
- Component by component
- Tạo wrapper component giữ API tương tự
- Parallel development (Tailwind components + Ant legacy)
- Gradual migration qua versions

### 5. Combine Tailwind + shadcn/ui + custom CSS?
**Trả lời:**
```jsx
// shadcn/ui đã dùng Tailwind, có thể mix bình thường
import { Button } from "@/components/ui/button"

export default function MyComponent() {
  return (
    <div className="flex gap-4 p-6 bg-gray-50">
      <Button className="w-32">Action</Button> {/* shadcn + Tailwind */}
      <custom-element className="custom-style" />
    </div>
  )
}
```

---

## 🎯 TIPS PHỎNG VẤN

### Điều phỏng vấn viên muốn nghe:
1. ✅ Hiểu rõ trade-off giữa 3 framework
2. ✅ Biết khi nào dùng cái nào
3. ✅ Có kinh nghiệm thực tế
4. ✅ Biết performance optimization
5. ✅ Có thể discuss architectural decisions

### Điều cần tránh:
1. ❌ "Tailwind là tốt nhất cho tất cả"
2. ❌ Không biết Ant Design bundle size
3. ❌ Không hiểu copy-paste model của shadcn
4. ❌ Quên custom component extraction
5. ❌ Không biết component composition

---

## 📝 PRACTICE CODE

### Task 1: Login Form với cả 3 framework
- Ant Design: Dùng Form component
- Tailwind: Viết CSS bằng class
- shadcn/ui: Dùng Form + Input components

### Task 2: Data Table
- Ant Design: `<Table />` component
- Tailwind: Tự build HTML table + styling
- shadcn/ui: Copy table component + tùy chỉnh

### Task 3: Theme Switching
- Ant Design: ConfigProvider
- Tailwind: Dark mode + color classes
- shadcn/ui: CSS variables

---

**Good luck với phỏng vấn! 🚀**
