#EduLead Pro - AI-Powered Lead Generation

A complete full-stack AI-powered lead generation web application for the education sector built with React.js, Node.js, Express.js, and TensorFlow.js.

#Features

- AI Lead Scoring - TensorFlow.js neural network for automatic lead quality prediction
- Admin Dashboard - Comprehensive analytics and lead management interface
- Public Lead Form - Student-facing form with real-time AI scoring
- Interactive Chatbot - AI-powered chatbot for lead qualification
- JWT Authentication - Secure admin access with token-based authentication
- CRM Integration - Mock Salesforce integration for high-quality leads
- Responsive Design - Modern UI with Tailwind CSS and shadcn/ui components
- Real-time Analytics - Live conversion tracking and lead statistics

#Tech Stack

#Frontend
- React 18 - Modern React with hooks and functional components
- TypeScript - Type-safe development
- Tailwind CSS - Utility-first CSS framework
- shadcn/ui - Modern component library
- TanStack Query - Data fetching and state management
- React Hook Form - Form validation and management
- Wouter - Lightweight routing
- Framer Motion - Animation library

#Backend
- Node.js - Runtime environment
- Express.js - Web framework
- TypeScript - Type-safe backend development
- JWT - Authentication and authorization
- bcryptjs - Password hashing
- Zod - Schema validation

### AI/ML
-TensorFlow.js - Neural network for lead scoring
-Custom AI Model - Trained on education sector patterns

#Database
-Drizzle ORM - Type-safe database queries
-In-Memory Storage - Development-ready storage layer
-PostgreSQL Ready - Production database support

edulead-pro
├── client/ # Frontend React application
│ ├── src/
│ │ ├── components/ # Reusable UI components
│ │ ├── hooks/ # Custom React hooks
│ │ ├── lib/ # Utility libraries and AI model
│ │ ├── pages/ # Application pages
│ │ └── App.tsx # Main application component
│ └── index.html # HTML entry point
├── server/ # Backend Node.js application
│ ├── index.ts # Server entry point
│ ├── routes.ts # API routes and endpoints
│ ├── storage.ts # Data storage layer
│ └── vite.ts # Development server setup
├── shared/ # Shared TypeScript schemas
│ └── schema.ts # Database and validation schemas
├── package.json # Dependencies and scripts
├── tsconfig.json # TypeScript configuration
├── vite.config.ts # Vite build configuration
├── tailwind.config.ts # Tailwind CSS configuration
└── postcss.config.js # PostCSS configuration
