Sure! Below is a comprehensive guide to building a marketing platform that enables advertisers and publishers to expand their business online. This guide will cover the project description, required features, tech stack selection, and a step-by-step coding example, including user authentication, roles, and testing.

### Project Description

**Marketing Platform Overview**:
The marketing platform connects advertisers with publishers, allowing them to create campaigns, track performance, and manage their accounts. Advertisers can create ads, set budgets, and monitor their campaigns, while publishers can display ads and earn revenue based on clicks or impressions.

### Required Features

1. **User Authentication**:
   - Sign up and login functionality.
   - Password hashing and secure storage.
   - JWT-based authentication for session management.

2. **User Roles**:
   - Different roles for users (Advertiser, Publisher, Admin).
   - Role-based access control to restrict certain actions.

3. **Campaign Management**:
   - Advertisers can create, update, and delete campaigns.
   - Publishers can view and manage ads displayed on their platforms.

4. **Analytics Dashboard**:
   - Basic analytics for advertisers to track campaign performance.
   - Revenue tracking for publishers.

5. **Database Management**:
   - Use PostgreSQL for data storage.
   - Use TypeORM for ORM functionality.

### Tech Stack Selection

- **Node.js**: For building the backend server.
- **Express**: A web framework for Node.js to handle routing and middleware.
- **PostgreSQL**: A powerful relational database for storing user and campaign data.
- **TypeORM**: An ORM for TypeScript and JavaScript that works with PostgreSQL.
- **Bcrypt**: For hashing passwords securely.
- **JWT (JSON Web Tokens)**: For secure user authentication.
- **Jest**: For testing the application.

### Step-by-Step Coding Example

#### Step 1: Set Up the Project

1. **Initialize the Project**:
   ```bash
   mkdir marketing-platform
   cd marketing-platform
   npm init -y
   ```

2. **Install Dependencies**:
   ```bash
   npm install express pg typeorm reflect-metadata bcryptjs jsonwebtoken dotenv
   npm install --save-dev typescript ts-node @types/node @types/express @types/bcryptjs @types/jsonwebtoken jest ts-jest @types/jest
   ```

3. **Set Up TypeScript**:
   Create a `tsconfig.json` file:
   ```json
   {
     "compilerOptions": {
       "target": "ES6",
       "module": "commonjs",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true,
       "outDir": "./dist"
     },
     "include": ["src/**/*"]
   }
   ```

4. **Set Up Jest**:
   Create a `jest.config.js` file:
   ```javascript
   module.exports = {
     preset: 'ts-jest',
     testEnvironment: 'node',
   };
   ```

5. **Create Directory Structure**:
   ```bash
   mkdir src
   mkdir src/controllers src/models src/routes src/middleware src/tests
   ```

#### Step 2: Create the Database Connection

Create a file `src/database.ts` for database connection:

```typescript
import { createConnection } from 'typeorm';
import { User } from './models/User';

export const connectDatabase = async () => {
  await createConnection({
    type: 'postgres',
    host: 'localhost',
    port: 5432,
    username: 'your_username',
    password: 'your_password',
    database: 'marketing_platform',
    entities: [User],
    synchronize: true,
  });
};
```

#### Step 3: Create User Model

Create a file `src/models/User.ts`:

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

export enum UserRole {
  ADVERTISER = 'advertiser',
  PUBLISHER = 'publisher',
  ADMIN = 'admin',
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

#### Step 4: Create Authentication Middleware

Create a file `src/middleware/auth.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export const authenticateJWT = (req: Request, res: Response, next: NextFunction) => {
  const token = req.header('Authorization')?.split(' ')[1];
  if (token) {
    jwt.verify(token, process.env.JWT_SECRET || 'your_jwt_secret', (err, user) => {
      if (err) {
        return res.sendStatus(403);
      }
      req.user = user;
      next();
    });
  } else {
    res.sendStatus(401);
  }
};
```

#### Step 5: Create User Controller

Create a file `src/controllers/userController.ts`:

```typescript
import { Request, Response } from 'express';
import { getRepository } from 'typeorm';
import { User, UserRole } from '../models/User';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';

export const registerUser = async (req: Request, res: Response) => {
  const { email, password, role } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const user = new User();
  user.email = email;
  user.password = hashedPassword;
  user.role = role || UserRole.ADVERTISER;

  const userRepository = getRepository(User);
  await userRepository.save(user);
  res.status(201).json({ message: 'User registered successfully' });
};

export const loginUser = async (req: Request, res: Response) => {
  const { email, password } = req.body;
  const userRepository = getRepository(User);
  const user = await userRepository.findOne({ where: { email } });

  if (user && (await bcrypt.compare(password, user.password))) {
    const token = jwt.sign({ id: user.id, role: user.role }, process.env.JWT_SECRET || 'your_jwt_secret', { expiresIn: '1h' });
    res.json({ token });
  } else {
    res.status(401).json({ message: 'Invalid credentials' });
  }
};
```

#### Step 6: Create Routes

Create a file `src/routes/userRoutes.ts`:

```typescript
import { Router } from 'express';
import { registerUser, loginUser } from '../controllers/userController';

const router = Router();

router.post('/register', registerUser);
router.post('/login', loginUser);

export default router;
```

#### Step 7: Set Up the Express Server

Create a file `src/server.ts`:

```typescript
import express from 'express';
import { connectDatabase } from './database';
import userRoutes from './routes/userRoutes';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
app.use(express.json());

connectDatabase()
  .then(() => {
    app.use('/api/users', userRoutes);
    app.listen(3000, () => {
      console.log('Server is running on http://localhost:3000');
    });
  })
  .catch((error) => console.log('Database connection error:', error));
```

#### Step 8: Create Tests

Create a file `src/tests/user.test.ts`:

```typescript
import request from 'supertest';
import { createConnection, getConnection } from 'typeorm';
import { User } from '../models/User';
import app from '../server'; // Adjust the import based on your server setup

describe('User Authentication', () => {
  beforeAll(async () => {
    await createConnection(); // Ensure the database connection is established
  });

  afterAll(async () => {
    await getConnection().close(); // Close the connection after tests
  });

  it('should register a new user', async () => {
    const response = await request(app)
      .post('/api/users/register')
      .send({ email: 'test@example.com', password: 'password123' });
    expect(response.status).toBe(201);
    expect(response.body.message).toBe('User registered successfully');
  });

  it('should login the user', async () => {
    const response = await request(app)
      .post('/api/users/login')
      .send({ email: 'test@example.com', password: 'password123' });
    expect(response.status).toBe(200);
    expect(response.body.token).toBeDefined();
  });
});
```

### Running the Application

1. **Create the Database**:
   Make sure you have PostgreSQL installed and create a database named `marketing_platform`.

2. **Set Up Environment Variables**:
   Create a `.env` file in the root of your project:
   ```
   JWT_SECRET=your_jwt_secret
   ```

3. **Run the Application**:
   To run the application in development mode, use:
   ```bash
   npx ts-node src/server.ts
   ```

   This command compiles and runs the TypeScript code directly without creating a build output. This is useful in development because it allows for quick iterations without the overhead of building the project each time.

4. **Run Tests**:
   To run the tests, use:
   ```bash
   npx jest
   ```

### Why Build Command is Not Used in Dev Mode

In development mode, using `ts-node` allows you to run TypeScript files directly without compiling them to JavaScript first. This speeds up the development process, as you can make changes and see the results immediately. The build command is typically used in production to compile TypeScript into optimized JavaScript files, which can then be served by a Node.js server.

### Conclusion

This guide provides a complete overview of building a marketing platform with user authentication, roles, and testing using TypeScript, PostgreSQL, and Jest. By following these steps, you can create a robust application that meets the needs of advertisers and publishers while ensuring secure and efficient user management. Feel free to expand on this foundation by adding more features and refining the application!
