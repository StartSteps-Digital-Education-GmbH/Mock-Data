### Mocking APIs and Data During Tests in a Marketing Platform with Jest and TypeScript

#### Overview

Mocking APIs allows developers to simulate responses for testing components in isolation, avoiding the need for live server connections. This approach is invaluable when testing services that rely on external dependencies or data, like campaign management, authentication, or analytics tracking in a marketing platform.

#### Key Benefits of Mocking

1. **Isolation**: Keeps tests focused on specific units, independent of other services or database states.
2. **Speed**: Reduces latency by removing network delays.
3. **Control**: Allows testing specific scenarios by defining exact responses.
4. **Simplification**: Minimizes test setup complexity, especially for state-sensitive APIs.

#### Setting Up Jest and Mocks in TypeScript

1. **Install Necessary Dependencies**:

   ```bash
   npm install --save-dev jest ts-jest @types/jest supertest @types/supertest
   ```

2. **Configure Jest for TypeScript**:

   Add a `jest.config.js` file to the root:

   ```javascript
   module.exports = {
     preset: 'ts-jest',
     testEnvironment: 'node',
   };
   ```

#### Example Project Structure

Assume a **Marketing Platform** backend structured as follows:

- **Controllers** (business logic): `src/controllers/`
- **Models** (data entities): `src/models/`
- **Routes**: `src/routes/`
- **Middleware**: `src/middleware/`
- **Tests**: `src/tests/`

#### Example Entities: `User` and `Campaign`

The **User** model represents an advertiser, publisher, or admin. The **Campaign** model represents ad campaigns created by advertisers.

- **User Model (User.ts)**:

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

- **Campaign Model (Campaign.ts)**:

   ```typescript
   import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

   @Entity()
   export class Campaign {
     @PrimaryGeneratedColumn()
     id: number;

     @Column()
     name: string;

     @Column()
     budget: number;

     @Column()
     status: string;  // E.g., 'active', 'paused', 'completed'
   }
   ```

#### Mocking API Responses in Controller Tests

We’ll create tests for the `CampaignController` and mock database calls.

1. **CampaignController (campaignController.ts)**:

   ```typescript
   import { Request, Response } from 'express';
   import { getRepository } from 'typeorm';
   import { Campaign } from '../models/Campaign';

   export const createCampaign = async (req: Request, res: Response) => {
     const { name, budget } = req.body;
     const campaignRepo = getRepository(Campaign);
     const campaign = campaignRepo.create({ name, budget, status: 'active' });
     await campaignRepo.save(campaign);
     res.status(201).json({ message: 'Campaign created', campaign });
   };
   ```

2. **Test Case for Campaign Creation (campaignController.test.ts)**

   Here, we mock the database operations to test the controller without connecting to a real database.

   ```typescript
   import { Request, Response } from 'express';
   import { createCampaign } from '../controllers/campaignController';
   import { Campaign } from '../models/Campaign';
   import { getRepository } from 'typeorm';

   jest.mock('typeorm', () => ({
     getRepository: jest.fn(),
   }));

   describe('CampaignController - createCampaign', () => {
     let req: Partial<Request>;
     let res: Partial<Response>;
     let saveMock: jest.Mock;

     beforeEach(() => {
       req = { body: { name: 'Test Campaign', budget: 1000 } };
       res = { json: jest.fn(), status: jest.fn().mockReturnThis() };

       saveMock = jest.fn();
       (getRepository as jest.Mock).mockReturnValue({ create: jest.fn(), save: saveMock });
     });

     it('should create a new campaign successfully', async () => {
       await createCampaign(req as Request, res as Response);

       expect(saveMock).toHaveBeenCalled();
       expect(res.status).toHaveBeenCalledWith(201);
       expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ message: 'Campaign created' }));
     });

     it('should handle errors', async () => {
       saveMock.mockRejectedValueOnce(new Error('Database error'));

       await expect(createCampaign(req as Request, res as Response)).rejects.toThrow('Database error');
     });
   });
   ```

#### Explanation of the Mock Setup

- **jest.mock**: Mocks the `typeorm` package, specifically the `getRepository` method.
- **Mocked Methods**: `create` and `save` functions within the `Campaign` repository are mocked to simulate database interactions.
- **Error Simulation**: A failing scenario for the `save` method is added by returning a rejected promise.

#### Running the Tests

To run the tests, execute:

```bash
npx jest
```

#### Advanced Mocking with `supertest`

For a more integrated approach, consider testing endpoints by combining `supertest` with a mocked server.

- **Example Endpoint Test with supertest (campaignEndpoints.test.ts)**:

   ```typescript
   import request from 'supertest';
   import app from '../server'; // Adjust to your app entry point

   describe('POST /api/campaigns', () => {
     it('should create a new campaign', async () => {
       const response = await request(app)
         .post('/api/campaigns')
         .send({ name: 'Test Campaign', budget: 500 });

       expect(response.status).toBe(201);
       expect(response.body).toHaveProperty('campaign');
     });
   });
   ```

### Conclusion

Mocking APIs in tests helps isolate the business logic, improve test reliability, and speed up development. By simulating the database interactions and API calls in your **Marketing Platform**, you create a more stable test environment, ensuring that your application performs reliably under various scenarios.
