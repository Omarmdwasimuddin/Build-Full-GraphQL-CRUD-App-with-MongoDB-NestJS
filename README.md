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


#### ``
```bash

```
---
