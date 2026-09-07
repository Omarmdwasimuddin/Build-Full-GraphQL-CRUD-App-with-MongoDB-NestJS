## Build Full GraphQL CRUD App with MongoDB & NestJS


#### Install
```bash
npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/express5 class-transformer class-validator graphql
```
---


#### Create module, service & resolver
```bash
nest g module book
```
```bash
nest g service book
```
```bash
nest g resolver book/resolvers/book --flat
```
---



>#### file-folder create koro- book/dto/create-book.input.ts , book/dto/update-book.input.ts & book/model/book.model.ts
>#### app.module.ts e add koro- imports: [ ConfigModule.forRoot({isGlobal: true,}), GraphQLModule.forRoot({driver: ApolloDriver,autoSchemaFile: join(process.cwd(), 'src/schema.gql'),sortSchema: true,playground: true,}), MongooseModule.forRoot(process.env.MONGO_URL!),]


#### `app.module.ts`
```bash
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { UserController } from './user/user.controller';
import { ProductService } from './product/product.service';
import { ProductController } from './product/product.controller';
import { MynameController } from './myname/myname.controller';
import { UserRolesController } from './user-roles/user-roles.controller';
import { ExceptionController } from './exception/exception.controller';
import { LoggerMiddleware } from './middleware/logger/logger.middleware';
import { DatabaseService } from './database/database.service';
import { DatabaseController } from './database/database.controller';
import { ConfigModule } from '@nestjs/config';
import { UserBdModule } from './user-bd/user-bd.module';
import { TypeOrmModule } from '@nestjs/typeorm';
import { EmployeeBdModule } from './employee-bd/employee-bd.module';
import { AuthModule } from './auth/auth.module';
import { MongooseModule } from '@nestjs/mongoose';
import { BookModule } from './book/book.module';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { join } from 'path';


@Module({
  imports: [ ConfigModule.forRoot({
    isGlobal: true,
  }), GraphQLModule.forRoot<ApolloDriverConfig>({
    driver: ApolloDriver,
    autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
    sortSchema: true,
    playground: true,
  }), TypeOrmModule.forRoot({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    autoLoadEntities: true,
    synchronize: true,
  }), MongooseModule.forRoot(process.env.MONGO_URL!), UserBdModule, EmployeeBdModule, AuthModule, BookModule],
  controllers: [AppController, UserController, ProductController, MynameController, UserRolesController, ExceptionController, DatabaseController ],
  providers: [AppService, ProductService, DatabaseService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer){
    consumer.apply(LoggerMiddleware).forRoutes('*');
  } 
}
```
---



#### `book.model.ts`
```bash
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';
import { ObjectType, Field, ID } from '@nestjs/graphql';

@Schema()
@ObjectType()
export class Book {
    @Field(() => ID)
    _id: string;

    @Prop({ required: true })
    @Field()
    title: string;

    @Prop()
    @Field({ nullable: true })
    description?: string;

    @Prop({ required: true })
    @Field()
    author: string;
}

export type BookDocument = HydratedDocument<Book>;
export const BookSchema = SchemaFactory.createForClass(Book);
```
---


#### `book.module.ts`
```bash
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
---


#### `create-book.input.ts`
```bash
import { InputType, Field } from "@nestjs/graphql";
import { IsNotEmpty, IsString } from "class-validator";

@InputType()
export class CreateBookInput {
    @Field()
    @IsString()
    @IsNotEmpty()
    title: string;

    @Field({ nullable: true })
    @IsString()
    description?: string;

    @Field()
    @IsString()
    @IsNotEmpty()
    author: string;
}
```
---


#### `update-book.input.ts`
```bash
import { CreateBookInput } from "./create-book.input";
import { InputType, Field, PartialType, ID } from "@nestjs/graphql";
import { IsNotEmpty } from "class-validator";

@InputType()
export class UpdateBookInput extends PartialType(CreateBookInput) {
    @Field(() => ID)
    @IsNotEmpty()
    id: string;
}
```
---


#### `book.service.ts`
```bash
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

    async delete(id: string): Promise<boolean> {
        const deletedBook = await this.bookModel.findByIdAndDelete(id);
        if (!deletedBook) throw new NotFoundException('Book not found!');
        return true;
    }
}
```
---


#### `book.resolver.ts`
```bash
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
    async delete(@Args('id', { type: () => String }) id: string) {
        return this.bookService.delete(id);
    }
}
```
---


#### visite localhost:3000/graphql
```bash
# localhost:3000/graphql
# mutation {
#  create(input: {
#    title: "Backend Framework",
#     description: "Nestja is joss...working!!",
#     author: "Wasim"
#   }){
#    _id,
#   title,
#  author
#   }
# }

# query {
#   getAllBooks {
#     _id,
#     title,
#     author
#   }
# }

# query {
#   getBook(id:"69992a9ac9fd36d1be436714" ) {
#     _id,
#     title,
#     author
#   }
# }

# mutation {
#   delete (id: "6999307ac9fd36d1be43671e")
# }


mutation {
  update(input:{
    id:"69992a5ec9fd36d1be436712",
    title: "GraphQL is awesome!",
    description:"joss!",
    author:"Wasim Uddin"
  }){
    _id,
    title,
    author
  }
}
```
---

>## OUTPUT
><img width="1599" height="811" alt="image" src="https://github.com/user-attachments/assets/4ccb28e9-4988-4142-bfc2-9c3ba74f5f8a" />
---
