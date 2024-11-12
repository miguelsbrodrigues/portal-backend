# Company Group Portal Backend Documentation
This document provides a comprehensive overview of the Company Group Portal Backend, detailing its purpose, architecture, setup instructions, testing strategies, and deployment guidelines.

## Table of Contents
## Project Overview
## Architecture
## Setup Instructions

# Project Overview
The Company Group Portal Backend is designed to manage and streamline various administrative and operational tasks within a company. It provides APIs and services to handle user management, role-based access control, data validation, and other core functionalities essential for company operations.

# Architecture
The backend is built using a modern stack to ensure scalability, maintainability, and performance. The key components include:

Next.js: A React-based framework that enables server-side rendering and static site generation, facilitating the development of web applications with enhanced performance and SEO capabilities.

Prisma: An ORM (Object-Relational Mapping) tool that simplifies database interactions and migrations, providing a type-safe database client.

PostgreSQL: A powerful, open-source relational database system known for its robustness and performance.

Cerbos: A scalable, open-source authorization layer that provides role-based access control (RBAC) to secure the application.

Zod: A TypeScript-first schema declaration and validation library, ensuring data integrity and type safety.

The application follows a modular architecture, separating concerns into distinct modules for user management, access control, and data handling. This structure enhances maintainability and scalability.

# Setup Instructions
## Steps to Start

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
├── package.json          # Project dependencies and scripts
├── config.js             # Centralized configuration file
└── README.md             # Project documentation
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

Open `schema.prisma` and define models for users, departments, and permissions:


  ```prisma
  model User {
  id              Int               @id @default(autoincrement())
  email           String            @unique
  password        String
  userCompanies   UserCompany[]     // Relationship with companies
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
}

model Company {
  id              Int               @id @default(autoincrement())
  name            String            @unique
  departments     Department[]
  userCompanies   UserCompany[]     // Users in this company
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
}

model Department {
  id              Int               @id @default(autoincrement())
  name            String
  companyId       Int
  company         Company           @relation(fields: [companyId], references: [id])
  userDepartments UserDepartment[]  // Users in this department
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
}

// Join table for User-Company relationship with role
model UserCompany {
  id              Int               @id @default(autoincrement())
  userId          Int
  companyId       Int
  roleId          Int
  user            User              @relation(fields: [userId], references: [id])
  company         Company           @relation(fields: [companyId], references: [id])
  role            Role              @relation(fields: [roleId], references: [id])
  userDepartments UserDepartment[]  // Departments within this company assignment
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt

  @@unique([userId, companyId])
}

// Join table for UserCompany-Department relationship with permissions
model UserDepartment {
  id              Int               @id @default(autoincrement())
  userCompanyId   Int
  departmentId    Int
  userCompany     UserCompany       @relation(fields: [userCompanyId], references: [id])
  department      Department        @relation(fields: [departmentId], references: [id])
  permissions     Permission[]      // Specific permissions for this department
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt

  @@unique([userCompanyId, departmentId])
}

model Role {
  id              Int               @id @default(autoincrement())
  name            String            @unique
  userCompanies   UserCompany[]
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
}

model Permission {
  id              Int               @id @default(autoincrement())
  action          String            // e.g., 'create', 'read', 'update', 'delete'
  resource        String            // e.g., 'employee_record', 'department', 'company'
  userDepartments UserDepartment[]
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt

  @@unique([action, resource])
}
  ```

- Create Initial Migrations for Department-Based Access Control:

  ```bash
  npx prisma migrate dev --name init_department_based_access
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
    // Create companies
    const company1 = await prisma.company.create({
      data: { name: 'Company A' }
    });
  
    const company2 = await prisma.company.create({
      data: { name: 'Company B' }
    });
  
    // Create departments for both companies
    const company1ITDept = await prisma.department.create({
      data: { name: 'IT', companyId: company1.id }
    });
  
    const company1HRDept = await prisma.department.create({
      data: { name: 'HR', companyId: company1.id }
    });
  
    const company2ITDept = await prisma.department.create({
      data: { name: 'IT', companyId: company2.id }
    });
  
    // Create roles
    const adminRole = await prisma.role.create({ data: { name: 'admin' } });
    const userRole = await prisma.role.create({ data: { name: 'user' } });
  
    // Create permissions
    const permissions = await Promise.all([
      prisma.permission.create({ data: { action: 'create', resource: 'employee_record' } }),
      prisma.permission.create({ data: { action: 'read', resource: 'employee_record' } }),
      prisma.permission.create({ data: { action: 'update', resource: 'employee_record' } }),
      prisma.permission.create({ data: { action: 'delete', resource: 'employee_record' } }),
    ]);
  
    // Create a user with different roles in different companies
    const user = await prisma.user.create({
      data: {
        email: 'john.doe@example.com',
        password: 'hashedpassword', // Remember to hash passwords in production
      }
    });
  
    // Assign user to Company A as admin
    const userCompanyA = await prisma.userCompany.create({
      data: {
        userId: user.id,
        companyId: company1.id,
        roleId: adminRole.id,
      }
    });
  
    // Assign user to Company B as regular user
    const userCompanyB = await prisma.userCompany.create({
      data: {
        userId: user.id,
        companyId: company2.id,
        roleId: userRole.id,
      }
    });
  
    // Assign department permissions for Company A
    await prisma.userDepartment.create({
      data: {
        userCompanyId: userCompanyA.id,
        departmentId: company1ITDept.id,
        permissions: {
          connect: permissions.map(p => ({ id: p.id })) // All permissions for IT
        }
      }
    });
  
    await prisma.userDepartment.create({
      data: {
        userCompanyId: userCompanyA.id,
        departmentId: company1HRDept.id,
        permissions: {
          connect: [{ id: permissions[1].id }] // Only read permission for HR
        }
      }
    });
  
    // Assign department permissions for Company B
    await prisma.userDepartment.create({
      data: {
        userCompanyId: userCompanyB.id,
        departmentId: company2ITDept.id,
        permissions: {
          connect: [{ id: permissions[1].id }] // Only read permission
        }
      }
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
  # cerbos/policies/employee_record.yaml
  version: "default"
  scope: "default"
  
  resource: "employee_record"
  
  vars:
    department_permissions: request.resource.attr.department_permissions
  
  rules:
    - actions: ["create", "read", "update", "delete"]
      roles: ["admin"]
      condition:
        match:
          expr: request.principal.attr.department_id == request.resource.attr.department_id
  
    - actions: ["read"]
      roles: ["user"]
      condition:
        match:
          expr: request.principal.attr.department_id == request.resource.attr.department_id
  ```

### 11. Integrate Cerbos Client

- Create a Cerbos client file (`lib/cerbos.js`) to connect to the Cerbos instance.

  **lib/cerbos.js**:
  ```javascript
  // lib/cerbos.js
  const { HTTP } = require('@cerbos/http');
  const cerbos = new HTTP('http://localhost:3592');
  
  module.exports = cerbos;
  
  // lib/accessControl.js
  const cerbos = require('./cerbos');
  const prisma = require('./prisma');
  
  async function checkAccess(user, resource, action, resourceId) {
    // Get department permissions
    const department = await prisma.department.findUnique({
      where: { id: user.departmentId },
      include: { permissions: true }
    });
  
    const decision = await cerbos.check({
      principal: {
        id: user.id.toString(),
        roles: [user.role.name],
        attr: {
          department_id: user.departmentId
        }
      },
      resource: {
        kind: resource,
        id: resourceId.toString(),
        attr: {
          department_id: department.id,
          department_permissions: department.permissions.map(p => p.action)
        }
      },
      actions: [action]
    });
  
    return decision.isAllowed(action);
  }
  
  module.exports = checkAccess;
  ```

- Add an access control utility (`lib/accessControl.js`) to check permissions.

  **lib/accessControl.js**:
  ```javascript
  // lib/accessControl.js
  const cerbos = require('./cerbos');
  const prisma = require('./prisma');
  
  async function checkAccess(userId, companyId, departmentId, resource, action) {
    // Get user's company association and role
    const userCompany = await prisma.userCompany.findUnique({
      where: {
        userId_companyId: {
          userId,
          companyId
        }
      },
      include: {
        role: true,
        userDepartments: {
          where: {
            departmentId
          },
          include: {
            permissions: true
          }
        }
      }
    });
  
    if (!userCompany || !userCompany.userDepartments?.[0]) {
      return false;
    }
  
    const department = userCompany.userDepartments[0];
    const decision = await cerbos.check({
      principal: {
        id: userId.toString(),
        roles: [userCompany.role.name],
        attr: {
          department_id: departmentId,
          company_id: companyId
        }
      },
      resource: {
        kind: resource,
        id: resource,
        attr: {
          department_id: departmentId,
          company_id: companyId,
          department_permissions: department.permissions.map(p => p.action)
        }
      },
      actions: [action]
    });
  
    return decision.isAllowed(action);
  }
  
  module.exports = checkAccess;
  ```

### 12. Apply Access Control in API Routes

- Use the `checkAccess` utility in your API routes to enforce RBAC.

  **pages/api/employee-records/[id].js**:
  ```javascript
  // pages/api/companies/[companyId]/departments/[departmentId]/records/[id].js
  import checkAccess from '../../../lib/accessControl';
  import prisma from '../../../lib/prisma';
  
  export default async function handler(req, res) {
    const { companyId, departmentId, id } = req.query;
    const user = req.user; // Get from your auth solution
  
    const hasAccess = await checkAccess(
      user.id,
      parseInt(companyId),
      parseInt(departmentId),
      'employee_record',
      'read'
    );
  
    if (!hasAccess) {
      return res.status(403).json({ message: 'Forbidden' });
    }
  
    // Proceed with the request
    const record = await prisma.employeeRecord.findFirst({
      where: {
        id: parseInt(id),
        departmentId: parseInt(departmentId),
        department: {
          companyId: parseInt(companyId)
        }
      }
    });
  
    if (!record) {
      return res.status(404).json({ message: 'Record not found' });
    }
  
    res.json(record);
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

6. Add Helper Functions

```javascript
// lib/userHelpers.js
async function getUserCompanyPermissions(userId, companyId) {
  return prisma.userCompany.findUnique({
    where: {
      userId_companyId: {
        userId,
        companyId
      }
    },
    include: {
      role: true,
      userDepartments: {
        include: {
          department: true,
          permissions: true
        }
      }
    }
  });
}

async function getAllUserCompanies(userId) {
  return prisma.userCompany.findMany({
    where: {
      userId
    },
    include: {
      company: true,
      role: true
    }
  });
}

module.exports = {
  getUserCompanyPermissions,
  getAllUserCompanies
};
   ```

