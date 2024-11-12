# 🏢 Company Group Portal Backend Setup Guide

This guide will help you set up the Company Group Portal Backend project, which uses **Next.js**, **Prisma**, **PostgreSQL**, **Cerbos** for Role-Based Access Control (RBAC), and **Zod** for data validation. Follow these steps to get started quickly.

## Steps to Start the Backend Project

### 1. Project Folder Structure

```plaintext
company-group-portal/
├── prisma/
│   ├── schema.prisma      # Prisma schema defining database models
│   ├── seed.js            # Script to seed the database with initial data
│   └── migrations/        # Directory containing migration files
├── cerbos/
│   └── policies/          # Directory containing Cerbos policy definitions
├── lib/
│   ├── cerbos.js          # Cerbos client setup
│   └── accessControl.js   # Access control utility
├── pages/
│   └── api/               # API routes for handling requests
├── .env                   # Environment variables
├── docker-compose.yml     # Docker Compose configuration
├── package.json           # Project dependencies and scripts
├── config.js              # Centralized configuration file
└── README.md              # Project documentation
```

### 2. Install Dependencies

- Initialize a new Next.js app and install required packages:

  ```bash
  npx create-next-app@latest .
  npm install prisma @prisma/client @cerbos/http zod bcrypt @prisma/zod-generator
  ```

### 3. Set Up Environment Variables

- Create a `.env` file in the root directory with the following variables:

  ```plaintext
  DATABASE_URL="postgresql://user:password@localhost:5432/company_portal"
  NEXTAUTH_SECRET="your-next-auth-secret"
  ```

- Replace `user`, `password`, and `company_portal` with your PostgreSQL credentials and database name.

### 4. Initialize Prisma

- Run Prisma initialization to set up your Prisma configuration:

  ```bash
  npx prisma init
  ```

### 5. Define Database Models

- Open `prisma/schema.prisma` and define models for users, roles, permissions, etc.

### 6. Create Initial Migrations for RBAC

- Define models for roles, users, and permissions in `schema.prisma`:

  ```prisma
  model User {
    id        Int      @id @default(autoincrement())
    email     String   @unique
    password  String
    roleId    Int
    role      Role     @relation(fields: [roleId], references: [id])
  }

  model Role {
    id          Int          @id @default(autoincrement())
    name        String       @unique
    permissions Permission[]
    users       User[]
  }

  model Permission {
    id      Int    @id @default(autoincrement())
    action  String
    resource String
    roles    Role[]
  }
  ```

- Run the initial migration command:

  ```bash
  npx prisma migrate dev --name init_rbac
  ```

### 7. Generate Additional Migrations

- Whenever you make changes to your database models, create a new migration to keep the database schema up to date:

  ```bash
  npx prisma migrate dev --name <migration_name>
  ```

### 8. Generate Zod Schemas from Prisma Models

- Add Zod generator to `schema.prisma`:
  ```prisma
  generator zod {
    provider = "prisma-zod-generator"
  }
  ```

- Run the Prisma generate command to create Zod schemas:
  ```bash
  npx prisma generate
  ```

### 9. Seed the Database

- Create a seed file (`prisma/seed.js`) to add initial data (roles, permissions, etc.).

  **prisma/seed.js**:
  ```javascript
  const { PrismaClient } = require('@prisma/client');
  const prisma = new PrismaClient();

  async function main() {
    const adminRole = await prisma.role.create({ data: { name: 'admin' } });
    const userRole = await prisma.role.create({ data: { name: 'user' } });

    const permissions = [
      { action: 'create', resource: 'post' },
      { action: 'read', resource: 'post' },
      { action: 'update', resource: 'post' },
      { action: 'delete', resource: 'post' },
    ];

    await prisma.permission.createMany({ data: permissions });

    await prisma.user.create({
      data: {
        email: 'admin@company.com',
        password: 'adminpassword',
        role: { connect: { id: adminRole.id } },
      },
    });
  }

  main()
    .catch((e) => {
      console.error(e);
      process.exit(1);
    })
    .finally(async () => {
      await prisma.$disconnect();
    });
  ```

- Run the seed script:

  ```bash
  npm run seed
  ```

### 10. Set Up Cerbos for RBAC

- Pull and run the Cerbos Docker container:

  ```bash
  docker run --name cerbos -p 3592:3592 ghcr.io/cerbos/cerbos
  ```

- Create a `cerbos/policies` folder and define your access policies.

  **cerbos/policies/post.yaml**:
  ```yaml
  version: "default"
  scope: "default"

  resource: "post"

  rules:
    - actions: ["create", "read", "update", "delete"]
      roles: ["admin"]
      effect: "allow"

    - actions: ["read"]
      roles: ["user"]
      effect: "allow"
  ```

### 11. Integrate Cerbos Client

- Create a Cerbos client file (`lib/cerbos.js`) to connect to the Cerbos instance.

  **lib/cerbos.js**:
  ```javascript
  const { HTTP } = require('@cerbos/http');
  const cerbos = new HTTP('http://localhost:3592');

  module.exports = cerbos;
  ```

- Add an access control utility (`lib/accessControl.js`) to check permissions.

  **lib/accessControl.js**:
  ```javascript
  const cerbos = require('./cerbos');

  async function checkAccess(user, resource, action) {
    const decision = await cerbos.check({
      principal: { id: user.id.toString(), roles: [user.role] },
      resource: { kind: resource, id: resource },
      actions: [action],
    });
    return decision.isAllowed(action);
  }

  module.exports = checkAccess;
  ```

### 12. Apply Access Control in API Routes

- Use the `checkAccess` utility in your API routes to enforce RBAC.

  **pages/api/posts/[id].js**:
  ```javascript
  import checkAccess from '../../../lib/accessControl';
  import prisma from '../../../lib/prisma';

  export default async function handler(req, res) {
    const { id } = req.query;
    const user = { id: req.user.id, role: req.user.role }; // Replace with actual user data

    const hasAccess = await checkAccess(user, 'post', 'update');
    if (!hasAccess) {
      return res.status(403).json({ message: 'Forbidden' });
    }

    if (req.method === 'PUT') {
      const updatedPost = await prisma.post.update({
        where: { id: parseInt(id, 10) },
        data: req.body,
      });
      res.json(updatedPost);
    } else {
      res.status(405).json({ message: 'Method not allowed' });
    }
  }
  ```

### 13. Validate Request Data Using Zod

- Import the generated Zod schemas into your API routes to validate incoming requests:

  **pages/api/users/[id].js**:
  ```javascript
  import { UserSchema } from '../../prisma/zod'; // Adjust the path as needed

  export default async function handler(req, res) {
    try {
      UserSchema.parse(req.body); // Validates request data against User model
    } catch (e) {
      return res.status(400).json({ error: e.errors });
    }

    // Proceed with handling the request
  }
  ```

### 14. Run the Development Server

- Start the Next.js development server:

  ```bash
  npm run dev
  ```

- Visit [http://localhost:3000](http://localhost:3000) to see the app in action.

### 15. Docker Compose Configuration

- Use Docker Compose to manage PostgreSQL and Cerbos easily. Create a `docker-compose.yml` file to define services for PostgreSQL and Cerbos.

  **docker-compose.yml**:
  ```yaml
  version: '3.9'

  services:
    postgres:
      image: postgres:13
      environment:
        POSTGRES_USER: user
        POSTGRES_PASSWORD: password
        POSTGRES_DB: company_portal
      ports:
        - "5432:5432"
      volumes:
        - postgres_data:/var/lib/postgresql/data

    cerbos:
      image: ghcr.io/cerbos/cerbos
      ports:
        - "3592:3592"
      volumes:
        - ./cerbos/policies:/policies

  volumes:
    postgres_data:
      driver: local
  ```

- Run Docker Compose to start services:

  ```bash
  docker-compose up -d
  ```

### 16. Development Strategy for Implementing an Endpoint

1. **Define Database Model (if needed)**: Add new models in `prisma/schema.prisma` and run migration.

2. **Create API Route**: Define the logic for handling requests in a new file in the `pages/api/` directory.

3. **Add Access Control**: Use `checkAccess` utility to enforce RBAC for the endpoint.

4. **Validate Request Data**: Use the generated Zod schemas to validate incoming request data.

5. **Test the Endpoint**: Use tools like **Postman** or **cURL** to test the endpoint.

### 17. Suggestions for Easier Development

1. **Use Configuration Templates**: Provide JSON/YAML templates for common configurations to speed up setup. For example, create templates for database settings, companies, and role definitions to simplify configuration.

2. **Centralized Configuration Management**: Use a `config.js` file to manage settings like database credentials, service URLs, and feature toggles. This reduces redundancy and makes it easier to update settings in one place.

   **Example `config.js`**:
   ```javascript
   module.exports = {
     database: {
       user: "db_user",
       password: "secure_password",
       databaseName: "company_portal",
     },
     services: {
       cerbosUrl: "http://localhost:3592",
       nextAuthSecret: "your-next-auth-secret",
     },
   };
   ```

3. **Automate Database Migrations and Seeding**: Add scripts in `package.json` to automate database migrations and seed data, making it easier for new developers to set up the environment quickly.

   **Example**:
   ```json
   {
     "scripts": {
       "migrate": "prisma migrate dev",
       "seed": "node prisma/seed.js"
     }
   }
   ```

4. **Use Docker for Local Development**: Use Docker to create consistent local development environments. Create a `docker-compose.yml` to spin up PostgreSQL and Cerbos, ensuring all developers have the same environment setup.

   **Example `docker-compose.yml`**:
   ```yaml
   version: '3.9'

   services:
     postgres:
       image: postgres:13
       environment:
         POSTGRES_USER: user
         POSTGRES_PASSWORD: password
         POSTGRES_DB: company_portal
       ports:
         - "5432:5432"
       volumes:
         - postgres_data:/var/lib/postgresql/data

     cerbos:
       image: ghcr.io/cerbos/cerbos
       ports:
         - "3592:3592"
       volumes:
         - ./cerbos/policies:/policies

   volumes:
     postgres_data:
       driver: local
   ```

5. **Dynamic Role and Company Management**: Create API routes to manage roles and companies dynamically, allowing for easy updates to user roles and company settings without direct database changes.

   **Example**:
   ```javascript
   import prisma from "../../../lib/prisma";

   export default async function handler(req, res) {
     if (req.method === "POST") {
       const { roleName, permissions } = req.body;
       const newRole = await prisma.role.create({
         data: {
           name: roleName,
           permissions: {
             create: permissions.map((permission) => ({ action: permission.action, resource: permission.resource })),
           },
         },
       });
       res.status(201).json(newRole);
     } else {
       res.status(405).json({ message: "Method not allowed" });
     }
   }
   ```

