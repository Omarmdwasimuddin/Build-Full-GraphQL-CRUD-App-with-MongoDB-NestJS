# Full GraphQL CRUD App with MongoDB & NestJS 

এতদিন REST API দিয়ে CRUD বানানো হয়েছে। এবার একই ধরনের কাজ **GraphQL** দিয়ে করা হবে — `Book` নামের একটা resource দিয়ে, MongoDB-এর সাথে। GraphQL-এ REST-এর মতো আলাদা আলাদা route থাকে না, বরং একটাই endpoint (`/graphql`) দিয়ে **query** (data পড়া) আর **mutation** (data পরিবর্তন) — দুই ধরনের operation চালানো যায়।

> এই গাইড শুরুর আগে MongoDB Atlas connect করা থাকতে হবে — দেখো: [Connect NestJS App with MongoDB Atlas](https://github.com/Omarmdwasimuddin/Connect-NestJS-App-with-MongoDB-Atlas)।

---

## ধাপ ১: প্রয়োজনীয় Package Install করা

```bash
npm i @nestjs/graphql@^13 @nestjs/apollo@^13 @apollo/server@^5 @as-integrations/express5 class-transformer class-validator graphql
```

### কোনটা কী কাজে লাগে

| Package | কাজ |
|---|---|
| `@nestjs/graphql` | NestJS-এ GraphQL integrate করার জন্য (Resolver, `@ObjectType`, `@Field` ইত্যাদি decorator) |
| `@nestjs/apollo` | Apollo Server-কে driver হিসেবে ব্যবহার করার জন্য |
| `@apollo/server` | মূল GraphQL server engine |
| `@as-integrations/express5` | Apollo Server-কে Express 5-এর সাথে যুক্ত করার adapter |
| `class-transformer`, `class-validator` | Input validation-এর জন্য (decorator-based, আগের Pipes গাইডে দেখানো হয়েছে) |
| `graphql` | মূল GraphQL library |

---

## ধাপ ২: Module, Service, Resolver তৈরি করা

```bash
nest g module book
```

```bash
nest g service book
```

```bash
nest g resolver book/resolvers/book --flat
```

> **নোট:** GraphQL-এ REST-এর "Controller"-এর জায়গায় থাকে **Resolver** — এটাই query/mutation handle করে।

এরপর manually এই folder/file গুলো বানাতে হবে:
- `book/dto/create-book.input.ts`
- `book/dto/update-book.input.ts`
- `book/model/book.model.ts`

---

## ধাপ ৩: `app.module.ts` Setup করা

> **নোট:** যোগ করতে হবে:
> ```ts
> imports: [
>   ConfigModule.forRoot({ isGlobal: true }),
>   GraphQLModule.forRoot({
>     driver: ApolloDriver,
>     autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
>     sortSchema: true,
>     playground: true,
>   }),
>   MongooseModule.forRoot(process.env.MONGO_URL!),
> ]
> ```

### `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { BookModule } from './book/book.module';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriverConfig, ApolloDriver } from '@nestjs/apollo';
import { join } from 'path';

@Module({
  imports: [ ConfigModule.forRoot({ isGlobal: true }), GraphQLModule.forRoot<ApolloDriverConfig>({
    driver: ApolloDriver,
    autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
    sortSchema: true,
    playground: true,
  }), MongooseModule.forRoot(process.env.MONGODB_URI!), BookModule ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### Option-গুলোর মানে

| Option | কাজ |
|---|---|
| `driver: ApolloDriver` | Apollo Server-কে GraphQL driver হিসেবে ব্যবহার করা |
| `autoSchemaFile` | Code-এ লেখা `@ObjectType`/`@Field` decorator থেকে automatic একটা `schema.gql` file জেনারেট করে দেয় (code-first approach) |
| `sortSchema: true` | জেনারেট হওয়া schema-র field/type গুলো alphabetically sort করে রাখে (readability-এর জন্য) |
| `playground: true` | Browser-এ `/graphql` URL-এ গেলে একটা interactive GraphQL playground UI চালু হয়, যেখানে query/mutation লিখে test করা যায় |

---

## ধাপ ৪: Model লেখা (Schema + GraphQL Type একসাথে)

### `book.model.ts`

```ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';
import { ObjectType, Field, ID } from '@nestjs/graphql';

@Schema()
@ObjectType()
export class Book {
    @Field(() => ID)
    _id!: string;

    @Prop({ required: true })
    @Field()
    title!: string;

    @Prop()
    @Field({ nullable: true })
    description?: string;

    @Prop({ required: true })
    @Field()
    author!: string;
}

export type BookDocument = HydratedDocument<Book>;
export const BookSchema = SchemaFactory.createForClass(Book);
```

**এখানে একটা গুরুত্বপূর্ণ জিনিস লক্ষ্য করার আছে:** একই class-এর উপরে দুটো decorator বসানো হয়েছে — `@Schema()` (Mongoose-এর জন্য) আর `@ObjectType()` (GraphQL-এর জন্য)। একইভাবে প্রতিটা property-তেও দুটো decorator — `@Prop()` (database column) আর `@Field()` (GraphQL field)। এর মানে **একই class দিয়ে একইসাথে database schema আর GraphQL type — দুটোই define হয়ে যাচ্ছে**, আলাদা করে duplicate করতে হচ্ছে না।

- `_id`-এ শুধু `@Field(() => ID)` আছে, `@Prop()` নেই — কারণ MongoDB নিজে থেকেই প্রতিটা document-এ `_id` বসিয়ে দেয়, আলাদা করে define করা লাগে না
- `description`-এ `@Field({ nullable: true })` — GraphQL schema-তে এই field optional (null হতে পারে) বলে বোঝানো হচ্ছে

---

## ধাপ ৫: Module-এ Model Register করা

### `book.module.ts`

```ts
import { Module } from '@nestjs/common';
import { BookService } from './book.service';
import { BookResolver } from './resolvers/book.resolver';
import { MongooseModule } from '@nestjs/mongoose';
import { Book, BookSchema } from './model/book.model';

@Module({
  imports: [MongooseModule.forFeature([{ name: Book.name, schema: BookSchema }])],
  providers: [BookService, BookResolver]
})
export class BookModule {}
```

লক্ষ্য করো: `controllers` array এখানে নেই (কারণ REST controller নেই), বরং `BookResolver`-কে `providers`-এ রাখা হয়েছে — GraphQL Resolver আসলে NestJS-এর কাছে একটা সাধারণ provider (`@Injectable()`-এর মতোই), controller না।

---

## ধাপ ৬: DTO (Input Type) লেখা

### `create-book.input.ts`

```ts
import { InputType, Field } from "@nestjs/graphql";
import { IsNotEmpty, IsString } from "class-validator";

@InputType()
export class CreateBookInput {
    @Field()
    @IsString()
    @IsNotEmpty()
    title!: string;

    @Field({ nullable: true })
    @IsString()
    description?: string;

    @Field()
    @IsString()
    @IsNotEmpty()
    author!: string;
}
```

GraphQL-এ REST-এর `@Body()` DTO-র জায়গায় থাকে **`@InputType()`** — mutation-এ argument হিসেবে যে data পাঠানো হবে, তার shape এখানে define করা হয়। `class-validator`-এর decorator (`@IsString()`, `@IsNotEmpty()`) এখানেও একইভাবে কাজ করে।

### `update-book.input.ts`

```ts
import { CreateBookInput } from "./create-book.input";
import { InputType, Field, PartialType, ID } from "@nestjs/graphql";
import { IsNotEmpty } from "class-validator";

@InputType()
export class UpdateBookInput extends PartialType(CreateBookInput) {
    @Field(() => ID)
    @IsNotEmpty()
    id!: string;
}
```

`PartialType(CreateBookInput)` extend করে — এর মানে `CreateBookInput`-এর সবগুলো field এখানে **automatic optional** হয়ে যায় (update করার সময় সবকিছু নতুন করে দেওয়া লাগে না, শুধু যেটা বদলাতে চাও সেটাই দিলেই হবে)। এর উপর নতুন করে একটা `id` field যোগ করা হয়েছে, যেটা required — কারণ কোন book update হবে সেটা তো বলতেই হবে।

---

## ধাপ ৭: Service লেখা (Business Logic)

### `book.service.ts`

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Book } from './model/book.model';
import { Model } from 'mongoose';
import { CreateBookInput } from './dto/create-book.input';
import { UpdateBookInput } from './dto/update-book.input';

@Injectable()
export class BookService {
    constructor(
        @InjectModel(Book.name) private bookModel: Model<Book>
    ) {}

    async create(input: CreateBookInput): Promise<Book> {
        const created = new this.bookModel(input);
        return created.save();
    }

    async findAll(): Promise<Book[]> {
        return this.bookModel.find().exec();
    }

    async findOne(id: string): Promise<Book> {
        const book = await this.bookModel.findById(id).exec();
        if (!book) throw new NotFoundException('Book not found!')
        return book;
    }

    async update(input: UpdateBookInput): Promise<Book> {
        const existingBook = await this.bookModel.findById(input.id);
        if (!existingBook) throw new NotFoundException('Book not found!');

        Object.assign(existingBook, input);
        return existingBook.save();
    }

    async remove(id: string): Promise<boolean> {
        const removedBook = await this.bookModel.findByIdAndDelete(id);
        if (!removedBook) throw new NotFoundException('Book not found!');
        return true;
    }
}
```

এই logic মূলত আগের REST CRUD গাইডগুলোর মতোই — `create`, `findAll`, `findOne`, `update` (`Object.assign` দিয়ে existing document-এর উপর নতুন data বসানো), আর `remove`। পার্থক্য শুধু input আসছে GraphQL Input Type থেকে, REST body থেকে না।

> **ছোট্ট একটা লক্ষণীয় বিষয়:** `update()`-এ `input`-এর ভিতরে থাকা `id` field-টাও `Object.assign()` দিয়ে document-এ বসে যায়। যেহেতু Mongoose schema-তে `id` নামে আলাদা কোনো `@Prop()` নেই (শুধু `_id` আছে), এই extra `id` property database-এ আলাদা field হিসেবে save হবে না — এটা নিরাপদ, কিন্তু চাইলে `update()`-এ `Object.assign()` করার আগে `input`-এর `id` field বাদ দিয়ে (destructure করে) পরিষ্কার রাখা আরেকটু ভালো practice হতো।

---

## ধাপ ৮: Resolver লেখা (Query/Mutation)

### `book.resolver.ts`

```ts
import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { BookService } from '../book.service';
import { Book } from '../model/book.model';
import { CreateBookInput } from '../dto/create-book.input';
import { UpdateBookInput } from '../dto/update-book.input';

@Resolver(() => Book)
export class BookResolver {
    constructor(
        private readonly bookService: BookService
    ) {}

    @Query(() => [Book], { name: "getAllBooks" })
    async findAll() {
        return this.bookService.findAll();
    }

    @Query(() => Book, { name: "getBook" })
    async findOne(@Args('id', { type: () => String }) id: string) {
        return this.bookService.findOne(id);
    }

    @Mutation(() => Book)
    async create(@Args('input') input: CreateBookInput) {
        return this.bookService.create(input);
    }

    @Mutation(() => Book)
    async update(@Args('input') input: UpdateBookInput) {
        return this.bookService.update(input);
    }

    @Mutation(() => Boolean)
    async remove(@Args('id', { type: () => String }) id: string) {
        return this.bookService.remove(id);
    }
}
```

### `@Query` vs `@Mutation`

- **`@Query()`** — data পড়ার (read) জন্য, যেমন REST-এর `GET`-এর সমতুল্য
- **`@Mutation()`** — data পরিবর্তনের (create/update/delete) জন্য, REST-এর `POST`/`PUT`/`DELETE`-এর সমতুল্য

### গুরুত্বপূর্ণ বিষয়

- `@Resolver(() => Book)` — বলে দিচ্ছে এই resolver `Book` type-এর সাথে সম্পর্কিত
- `{ name: "getAllBooks" }` — GraphQL schema-তে query-র নাম কী হবে সেটা override করা হচ্ছে (method-এর নাম `findAll`, কিন্তু GraphQL-এ এটা `getAllBooks` নামে দেখা যাবে)
- `@Args('id', { type: () => String })` — GraphQL query/mutation-এ argument হিসেবে `id` নেওয়া হচ্ছে, type explicit করে বলে দেওয়া হয়েছে `String`
- `@Args('input')` — পুরো `CreateBookInput`/`UpdateBookInput` object-টাই `input` নামের একটা argument হিসেবে নেওয়া হচ্ছে

---

## ধাপ ৯: GraphQL Playground-এ Test করা

`localhost:3000/graphql`-এ গিয়ে নিচের query/mutation-গুলো try করা যায়।

### Book তৈরি করা (Create)

```graphql
mutation {
  create(input: {
    title: "NestJS is awesome!",
    description: "This is awesome framework for backend developing....",
    author: "Md Wasim Uddin"
  }) {
    _id,
    title,
    description,
    author
  }
}
```

### সবগুলো Book দেখা (Read - All)

```graphql
query {
  getAllBooks {
    _id,
    title,
    author
  }
}
```

### নির্দিষ্ট একটা Book দেখা (Read - One)

```graphql
query {
  getBook(id: "6a9f92b35daf41f4cb80e19f") {
    _id,
    title,
    author
  }
}
```

### Book Update করা

```graphql
mutation {
  update(input: {
    id: "6a9f92b35daf41f4cb80e19f",
    title: "GraphQL is awesome!",
    description: "joss!",
    author: "Wasim Uddin"
  }) {
    _id,
    title,
    author
  }
}
```

### Book মুছে ফেলা (Delete)

```graphql
mutation {
  remove(id: "6a9f92b35daf41f4cb80e19f")
}
```

> **লক্ষ্য করো:** GraphQL-এর প্রতিটা query/mutation-এ `{ }`-এর ভিতরে বলে দিতে হয় response-এ কোন কোন field ফেরত চাও (এখানে `_id`, `title`, `author` ইত্যাদি) — এটাই GraphQL-এর মূল সুবিধা: শুধু যেটা দরকার সেটাই fetch হয়, over-fetching হয় না।

---

## Output (উদাহরণ)

GraphQL Playground-এ query চালিয়ে যেমন response আসে:

![GraphQL Playground output](https://github.com/user-attachments/assets/4ccb28e9-4988-4142-bfc2-9c3ba74f5f8a)

---
