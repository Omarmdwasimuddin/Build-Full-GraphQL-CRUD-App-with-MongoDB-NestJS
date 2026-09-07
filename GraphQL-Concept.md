# GraphQL — Concept Guide

## 1. GraphQL কী?

**GraphQL হলো API তৈরি ও data request করার একটি query language এবং API specification।**

এটি database নয়।

```text
Frontend
   ↓
REST API / GraphQL API
   ↓
NestJS
   ↓
MongoDB / PostgreSQL / MySQL
```

মনে রাখবে:

- MongoDB = Database
- REST = API তৈরির একটি পদ্ধতি
- GraphQL = API তৈরির আরেকটি পদ্ধতি
- NestJS = Backend framework

---

## 2. GraphQL কেন ব্যবহার করা হয়?

GraphQL-এর মূল সুবিধা হলো:

> **Client নিজেই বলে দিতে পারে সে কোন data এবং কোন field চায়।**

ধরো Student-এর data:

```text
id
name
email
age
address
phone
```

Client যদি শুধু `name` এবং `email` চায়:

```graphql
query {
  student(id: 1) {
    name
    email
  }
}
```

Response-এ প্রয়োজনীয় field-গুলোই আসবে।

এতে **over-fetching** কমে এবং client-এর data fetching বেশি flexible হয়।

---

## 3. REST বনাম GraphQL

### REST

সাধারণত আলাদা আলাদা endpoint থাকে:

```text
GET    /students
GET    /students/1
POST   /students
PATCH  /students/1
DELETE /students/1
```

### GraphQL

সাধারণত একটি endpoint ব্যবহার করা হয়:

```text
/graphql
```

তারপর query বা mutation-এর মাধ্যমে client বলে দেয় কী করতে হবে এবং কী data দরকার।

---

## 4. GraphQL-এর প্রধান ৩টি operation

### Query

Data **পড়ার** জন্য।

```graphql
query {
  students {
    id
    name
    email
  }
}
```

REST-এর `GET`-এর সাথে তুলনা করা যায়।

### Mutation

Data **create, update বা delete** করার জন্য।

```graphql
mutation {
  createStudent(name: "Wasim", email: "wasim@gmail.com") {
    id
    name
    email
  }
}
```

REST-এর `POST`, `PATCH`, `DELETE`-এর সাথে তুলনা করা যায়।

### Subscription

**Real-time data** পাওয়ার জন্য।

যেমন chat বা live notification।

---

## 5. Schema কী?

**Schema হলো GraphQL API-এর contract বা structure।**

এটি বলে দেয় API-তে কী ধরনের data আছে এবং কীভাবে data পাওয়া/পরিবর্তন করা যাবে।

উদাহরণ:

```graphql
type Student {
  id: ID!
  name: String!
  email: String!
  age: Int
}
```

এখানে:

- `Student` = একটি GraphQL type
- `id`, `name`, `email`, `age` = field
- `String`, `Int`, `ID` = type
- `!` = required / non-null

---

## 6. Resolver কী?

**Resolver হলো GraphQL-এর request handler।**

REST-এ যেমন:

```text
Controller → Service → Database
```

GraphQL-এ:

```text
Resolver → Service → Database
```

NestJS-এ:

```typescript
@Resolver(() => Student)
export class StudentResolver {

  constructor(
    private readonly studentService: StudentService,
  ) {}

  @Query(() => [Student])
  students() {
    return this.studentService.findAll();
  }
}
```

এখানে `@Query()` GraphQL query handle করছে এবং `StudentService` থেকে data নিচ্ছে।

---

## 7. NestJS-এ REST ও GraphQL-এর mapping

```text
REST                    GraphQL

Controller       →      Resolver
GET              →      Query
POST             →      Mutation
PATCH            →      Mutation
DELETE            →      Mutation
DTO/Input        →      Input Type
Response model   →      Object Type
```

সবকিছু একদম identical নয়, তবে concept বোঝার জন্য এই mapping useful।

---

## 8. GraphQL কখন ব্যবহার করব?

GraphQL বেশি useful যখন:

- Frontend-এর data requirement অনেক flexible
- একই resource-এর বিভিন্ন জায়গায় বিভিন্ন field দরকার
- অনেক nested/related data আছে
- Web, mobile ইত্যাদি multiple clients আছে
- এক query-তে related data নেওয়া সুবিধাজনক
- Client-side data fetching-এর উপর বেশি control দরকার

উদাহরণ:

```text
User
 ↓
Posts
 ↓
Comments
 ↓
Comment Author
```

এ ধরনের complex relationship থাকলে GraphQL useful হতে পারে।

---

## 9. কখন REST ব্যবহার করব?

যদি API simple CRUD হয় এবং data requirement straightforward হয়, তাহলে REST অনেক সময় ভালো choice।

যেমন:

```text
GET    /students
GET    /students/:id
POST   /students
PATCH  /students/:id
DELETE /students/:id
```

ছোট/মাঝারি CRUD project-এ অকারণে GraphQL যোগ করার প্রয়োজন নেই।

---

## 10. একই Project-এ REST + GraphQL?

**হ্যাঁ, একই project-এ দুটোই ব্যবহার করা যায়।**

Architecture হতে পারে:

```text
                 NestJS
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
     REST API             GraphQL API
          │                   │
          └─────────┬─────────┘
                    ↓
                 Service
                    ↓
                 Database
```

অর্থাৎ REST Controller এবং GraphQL Resolver একই Service ব্যবহার করতে পারে।

কোনো project-এর simple অংশ REST এবং complex/flexible data অংশ GraphQL দিয়েও করা সম্ভব।

---

## 11. REST নাকি GraphQL — কীভাবে সিদ্ধান্ত নেব?

Database দেখে সিদ্ধান্ত নেবে না।

```text
❌ MongoDB → GraphQL
❌ PostgreSQL → REST
```

বরং দেখবে:

```text
Client কীভাবে data ব্যবহার করবে?
        ↓
Data requirement কতটা complex?
        ↓
অনেক related/nested data আছে?
        ↓
Multiple clients আছে?
        ↓
Client-এর flexible data fetching দরকার?
```

### সহজ rule

```text
Simple API / CRUD
      ↓
    REST

Complex + Flexible data requirements
      ↓
   GraphQL

Project-এর কিছু অংশ simple,
কিছু অংশ complex
      ↓
 REST + GraphQL
```

---

# Quick Revision

```text
GraphQL
  ↓
API layer
  ↓
Client নিজের প্রয়োজনের field select করতে পারে
  ↓
Schema = API contract
  ↓
Resolver = request handler
  ↓
Query = read
Mutation = create/update/delete
Subscription = real-time
```

## সবচেয়ে গুরুত্বপূর্ণ কথা

> **GraphQL database-এর alternative নয়; GraphQL হলো API তৈরির একটি alternative approach, বিশেষ করে যখন client-এর data requirement complex ও flexible হয়।**
