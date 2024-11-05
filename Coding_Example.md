# Live Coding Example

### Platform Overview

**Marketing Platform Features:**
1. **User Authentication**: Users can register and log in.
2. **User Roles**: Different roles (Advertiser, Publisher) with specific permissions.
3. **PostgreSQL Database**: Store user data and roles.
4. **TypeORM**: ORM for interacting with the PostgreSQL database.
5. **Bcrypt**: For hashing passwords.
6. **JWT**: For secure authentication.
7. **Jest**: For testing the application.

### Requirements

- Node.js
- PostgreSQL
- TypeScript
- TypeORM
- Bcrypt
- JWT
- Jest

### Step-by-Step Implementation

#### Step 1: Set Up the Project

1. **Create a new directory for your project**:

   ```bash
   mkdir marketing-platform
   cd marketing-platform
   ```

2. **Initialize a new Node.js project**:

   ```bash
   npm init -y
   ```

3. **Install required dependencies**:

   ```bash
   npm install express typeorm pg bcrypt jsonwebtoken dotenv
   ```

4. **Install development dependencies**:

   ```bash
   npm install --save-dev typescript ts-node @types/node @types/express jest ts-jest @types/jest
   ```

5. **Initialize TypeScript**:

   ```bash
   npx tsc --init
   ```

6. **Create a `.env` file** for environment variables:

   ```plaintext
   DATABASE_URL=postgres://user:password@localhost:5432/marketing_platform
   JWT_SECRET=your_jwt_secret
   ```

#### Step 2: Set Up TypeORM and PostgreSQL

1. **Create a `ormconfig.json` file**:

   ```json
   {
     "type": "postgres",
     "host": "localhost",
     "port": 5432,
     "username": "user",
     "password": "password",
     "database": "marketing_platform",
     "synchronize": true,
     "logging": false,
     "entities": ["src/entity/**/*.ts"]
   }
   ```

2. **Create the User entity**:

   Create a directory `src/entity` and a file `User.ts`:

   ```typescript
   // src/entity/User.ts
   import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

   export enum UserRole {
     ADVERTISER = 'advertiser',
     PUBLISHER = 'publisher',
   }

   @Entity()
   export class User {
     @PrimaryGeneratedColumn()
     id: number;

     @Column({ unique: true })
     email: string;

     @Column()
     password: string;

     @Column({
       type: 'enum',
       enum: UserRole,
       default: UserRole.ADVERTISER,
     })
     role: UserRole;
   }
   ```

#### Step 3: Create Authentication Logic

1. **Create an `auth.ts` file** in the `src` directory:

   ```typescript
   // src/auth.ts
   import express from 'express';
   import { getRepository } from 'typeorm';
   import { User, UserRole } from './entity/User';
   import bcrypt from 'bcrypt';
   import jwt from 'jsonwebtoken';

   const router = express.Router();

   // Register
   router.post('/register', async (req, res) => {
     const { email, password, role } = req.body;
     const hashedPassword = await bcrypt.hash(password, 10);
     const user = getRepository(User).create({ email, password: hashedPassword, role });
     await getRepository(User).save(user);
     res.status(201).send('User registered');
   });

   // Login
   router.post('/login', async (req, res) => {
     const { email, password } = req.body;
     const user = await getRepository(User).findOne({ where: { email } });
     if (!user) return res.status(404).send('User not found');

     const isMatch = await bcrypt.compare(password, user.password);
     if (!isMatch) return res.status(401).send('Invalid credentials');

     const token = jwt.sign({ id: user.id, role: user.role }, process.env.JWT_SECRET!, { expiresIn: '1h' });
     res.json({ token });
   });

   export default router;
   ```

#### Step 4: Set Up the Express Server

1. **Create an `index.ts` file** in the `src` directory:

   ```typescript
   // src/index.ts
   import 'reflect-metadata';
   import express from 'express';
   import { createConnection } from 'typeorm';
   import authRoutes from './auth';
   import dotenv from 'dotenv';

   dotenv.config();

   const app = express();
   app.use(express.json());
   app.use('/auth', authRoutes);

   createConnection()
     .then(() => {
       app.listen(3000, () => {
         console.log('Server is running on http://localhost:3000');
       });
     })
     .catch((error) => console.log(error));
   ```

#### Step 5: Create Tests with Jest

1. **Create a `__tests__` directory** in the `src` directory and a file `auth.test.ts`:

   ```typescript
   // src/__tests__/auth.test.ts
   import request from 'supertest';
   import { createConnection, getConnection } from 'typeorm';
   import app from '../index'; // Adjust the import based on your structure

   describe('Auth API', () => {
     beforeAll(async () => {
       await createConnection();
     });

     afterAll(async () => {
       await getConnection().close();
     });

     it('should register a new user', async () => {
       const response = await request(app)
         .post('/auth/register')
         .send({ email: 'test@example.com', password: 'password', role: 'advertiser' });
       expect(response.status).toBe(201);
     });

     it('should log in a user', async () => {
       const response = await request(app)
         .post('/auth/login')
         .send({ email: 'test@example.com', password: 'password' });
       expect(response.status).toBe(200);
       expect(response.body.token).toBeDefined();
     });
   });
   ```

#### Step 6: Running the Application

1. **Run the application in development mode**:

   You can run the application using `ts-node`:

   ```bash
   npx ts-node src/index.ts
   ```

   This command allows you to run TypeScript files directly without compiling them to JavaScript first. This is useful in development mode because it speeds up the development process by eliminating the need for a build step.

2. **Run the tests**:

   To run the tests, you can use:

   ```bash
   npx jest
   ```

### Why Build Command is Not Used in Dev Mode

In development mode, using `ts-node` allows for rapid iteration without the overhead of compiling TypeScript to JavaScript. This is particularly useful for testing and debugging. The build command is typically used in production to compile TypeScript files into optimized JavaScript files, which can then be run in a Node.js environment.

### Conclusion

You now have a basic marketing platform with user authentication, roles, and testing capabilities. This example covers the essential components and provides a solid foundation for further development. You can expand this platform by adding more features, such as managing campaigns, tracking performance, and integrating with third-party services.
