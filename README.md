### 1. Структура проекту (Monorepo підхід)
```text
employee-management-system/
├── client/          # Frontend (Vite + React)
├── server/          # Backend (Next.js API)
├── .gitignore
└── README.md
```
### 2. Поетапне створення проекту
#### Крок 1: Ініціалізація проекту
```text
mkdir employee-management-system
cd employee-management-system
pnpm init
```
#### Крок 2: Налаштування клієнтської частини (React)
```markdown
pnpm create vite@latest client --template react-ts
cd client
pnpm install @reduxjs/toolkit react-redux @types/react-redux react-router-dom date-fns axios
pnpm install -D @types/react-router-dom
```
#### Структура клієнта:
```text
client/
├── public/
├── src/
│   ├── api/                  # RTK Query endpoints
│   ├── components/
│   │   ├── auth/
│   │   ├── employees/
│   │   ├── ui/
│   ├── pages/
│   │   ├── AdminPage.tsx
│   │   ├── AuthPage.tsx
│   │   ├── EmployeePage.tsx
│   │   ├── SettingsPage.tsx
│   │   ├── AddEmployeePage.tsx
│   ├── store/                # Redux store
│   ├── types/                # TypeScript типи
│   ├── utils/                # Допоміжні функції
│   ├── App.tsx
│   ├── main.tsx
├── index.html
├── package.json
├── tsconfig.json
```
#### Крок 3: Налаштування серверної частини (Next.js API)
```text
cd ..
pnpm create next-app server --typescript --eslint
cd server
pnpm install mongoose bcryptjs jsonwebtoken swagger-ui-express swagger-jsdoc cors
pnpm install -D @types/bcryptjs @types/jsonwebtoken @types/cors
```
#### Структура сервера:
```text
server/
├── pages/
│   ├── api/
│   │   ├── auth/
│   │   │   └── login.ts
│   │   ├── employees/
│   │   │   ├── index.ts      # GET/POST
│   │   │   └── [id].ts       # GET/PUT/DELETE
│   │   ├── settings/
│   │   │   └── index.ts
│   │   └── swagger.ts        # Swagger документація
├── models/
│   ├── Employee.ts           # Mongoose модель
│   ├── Settings.ts
│   ├── User.ts               # Для адмінів
├── lib/
│   ├── dbConnect.ts          # Підключення до MongoDB
│   └── middleware.ts         # Auth middleware
├── package.json
├── tsconfig.json
└── next.config.js
```
#### Крок 4: Підключення MongoDB
1. Створіть безкоштовний кластер на Atlas MongoDB
2. Додайте підключення в server/lib/dbConnect.ts:
```ts
import mongoose from 'mongoose';

const MONGODB_URI = process.env.MONGODB_URI || '';

if (!MONGODB_URI) {
  throw new Error('Please define the MONGODB_URI environment variable');
}

let cached = global.mongoose;

if (!cached) {
  cached = global.mongoose = { conn: null, promise: null };
}

async function dbConnect() {
  if (cached.conn) return cached.conn;

  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI).then((mongoose) => mongoose);
  }

  cached.conn = await cached.promise;
  return cached.conn;
}

export default dbConnect;
```
#### Крок 5: Моделі для MongoDB
##### Приклад моделі працівника (server/models/Employee.ts):
```ts
import { Schema, model, Document } from 'mongoose';

interface IEmployee extends Document {
  name: string;
  position: 'seller' | 'admin' | 'manager' | 'courier';
  baseRate: number;
  salesPercentage: number;
  workDays: {
    date: Date;
    shop: string;
    sales: number;
    penalties: number;
    notes: string;
  }[];
}

const EmployeeSchema = new Schema<IEmployee>({
  name: { type: String, required: true },
  position: { type: String, required: true },
  baseRate: { type: Number, required: true },
  salesPercentage: { type: Number, required: true },
  workDays: [{
    date: { type: Date, default: Date.now },
    shop: String,
    sales: { type: Number, default: 0 },
    penalties: { type: Number, default: 0 },
    notes: String
  }]
});

export const Employee = model<IEmployee>('Employee', EmployeeSchema);
```
#### Крок 6: API Endpoints
##### Приклад endpoint для працівників (server/pages/api/employees/index.ts):
```ts
import { NextApiRequest, NextApiResponse } from 'next';
import dbConnect from '../../../lib/dbConnect';
import { Employee } from '../../../models/Employee';
import { authMiddleware } from '../../../lib/middleware';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  await dbConnect();
  await authMiddleware(req, res);

  switch (req.method) {
    case 'GET':
      try {
        const employees = await Employee.find({});
        res.status(200).json(employees);
      } catch (error) {
        res.status(500).json({ message: 'Server error' });
      }
      break;
    case 'POST':
      try {
        const employee = new Employee(req.body);
        await employee.save();
        res.status(201).json(employee);
      } catch (error) {
        res.status(400).json({ message: 'Invalid data' });
      }
      break;
    default:
      res.setHeader('Allow', ['GET', 'POST']);
      res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```
#### Крок 7: Налаштування RTK Query
##### Приклад API slice (client/src/api/employeesApi.ts):
```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const employeesApi = createApi({
  reducerPath: 'employeesApi',
  baseQuery: fetchBaseQuery({ 
    baseUrl: '/api/',
    prepareHeaders: (headers) => {
      const token = localStorage.getItem('token');
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    }
  }),
  endpoints: (builder) => ({
    getEmployees: builder.query({
      query: () => 'employees',
    }),
    addEmployee: builder.mutation({
      query: (employee) => ({
        url: 'employees',
        method: 'POST',
        body: employee,
      }),
    }),
  }),
});

export const { useGetEmployeesQuery, useAddEmployeeMutation } = employeesApi;
```
#### Крок 8: Налаштування маршрутизації
##### Приклад App.tsx:
```ts
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthPage, AdminPage, EmployeePage, SettingsPage, AddEmployeePage } from './pages';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<AuthPage />} />
        <Route path="/admin" element={<AdminPage />} />
        <Route path="/employee/:id" element={<EmployeePage />} />
        <Route path="/settings" element={<SettingsPage />} />
        <Route path="/add-employee" element={<AddEmployeePage />} />
      </Routes>
    </BrowserRouter>
  );
}
```
#### Крок 9: Запуск проекту
##### 1. В одному терміналі:
```text
cd server
pnpm dev
```
##### 2. В іншому терміналі:
```text
cd client
pnpm dev
```
#### Крок 10: Деплой
##### 1. Клієнт на Vercel:
- Імпортуйте папку client як Vite проект
- Вкажіть build команду: pnpm build
##### 2. Сервер на Vercel:
- Імпортуйте папку server як Next.js проект
- Додайте змінні оточення (MONGODB_URI, JWT_SECRET)

#### Додаткові поради:
1. Для авторизації використовуйте JWT токени
2. Для календаря подивіться react-big-calendar або аналоги
3. Для таблиць - react-table v8
4. Для стилів - Tailwind CSS або MUI
5. Для перевірки API використовуйте Postman або Swagger UI

Важливо: перед деплоєм налаштуйте CORS на сервері та перевірте всі environment variables.
Це базова структура, яку ви можете адаптувати під свої потреби. Головне - починайте з простих реалізацій кожної функціональності і поступово ускладнюйте.