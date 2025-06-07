#shared/schema.ts
```typescript
import { pgTable, text, serial, integer, boolean, timestamp, jsonb } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

// Database table definitions
export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  username: text("username").notNull().unique(),
  password: text("password").notNull(),
  role: text("role").notNull().default("admin"),
  createdAt: timestamp("created_at").defaultNow(),
});

export const leads = pgTable("leads", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull(),
  phone: text("phone"),
  age: integer("age"),
  location: text("location"),
  interest: text("interest").notNull(),
  degreeLevel: text("degree_level"),
  timeline: text("timeline"),
  comments: text("comments"),
  score: integer("score").default(0),
  quality: text("quality").default("medium"),
  status: text("status").default("new"),
  aiPrediction: jsonb("ai_prediction"),
  crmSynced: boolean("crm_synced").default(false),
  crmId: text("crm_id"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const chatSessions = pgTable("chat_sessions", {
  id: serial("id").primaryKey(),
  sessionId: text("session_id").notNull().unique(),
  messages: jsonb("messages").default([]),
  leadData: jsonb("lead_data"),
  createdAt: timestamp("created_at").defaultNow(),
});

// Validation schemas
export const insertUserSchema = createInsertSchema(users).omit({
  id: true,
  createdAt: true,
});

export const insertLeadSchema = createInsertSchema(leads).omit({
  id: true,
  createdAt: true,
  updatedAt: true,
  crmSynced: true,
  crmId: true,
}).extend({
  age: z.number().min(16).max(100).optional(),
  score: z.number().min(0).max(100).optional(),
});

export const insertChatSessionSchema = createInsertSchema(chatSessions).omit({
  id: true,
  createdAt: true,
});

export const updateLeadStatusSchema = z.object({
  status: z.enum(["new", "contacted", "qualified", "converted"]),
});

export const loginSchema = z.object({
  username: z.string().min(3),
  password: z.string().min(6),
});

// Type exports
export type InsertUser = z.infer<typeof insertUserSchema>;
export type User = typeof users.$inferSelect;
export type InsertLead = z.infer<typeof insertLeadSchema>;
export type Lead = typeof leads.$inferSelect;
export type InsertChatSession = z.infer<typeof insertChatSessionSchema>;
export type ChatSession = typeof chatSessions.$inferSelect;
export type UpdateLeadStatus = z.infer<typeof updateLeadStatusSchema>;
export type LoginRequest = z.infer<typeof loginSchema>;
```

#server/index.ts
```typescript
import express, { type Request, Response, NextFunction } from "express";
import { registerRoutes } from "./routes";
import { setupVite, serveStatic, log } from "./vite";

const app = express();

// Middleware setup
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// Request logging middleware
app.use((req, res, next) => {
  const start = Date.now();
  const path = req.path;
  let capturedJsonResponse: Record<string, any> | undefined = undefined;

  const originalResJson = res.json;
  res.json = function (bodyJson, ...args) {
    capturedJsonResponse = bodyJson;
    return originalResJson.apply(res, [bodyJson, ...args]);
  };

  res.on("finish", () => {
    const duration = Date.now() - start;
    if (path.startsWith("/api")) {
      let logLine = `${req.method} ${path} ${res.statusCode} in ${duration}ms`;
      if (capturedJsonResponse) {
        logLine += ` :: ${JSON.stringify(capturedJsonResponse).substring(0, 100)}`;
      }
      log(logLine);
    }
  });

  next();
});

// Initialize server
(async () => {
  const server = await registerRoutes(app);

  // Global error handling
  app.use((err: any, _req: Request, res: Response, _next: NextFunction) => {
    const status = err.status || err.statusCode || 500;
    const message = err.message || "Internal Server Error";
    res.status(status).json({ message });
    throw err;
  });

  // Setup development or production serving
  if (app.get("env") === "development") {
    await setupVite(app, server);
  } else {
    serveStatic(app);
  }

  const PORT = process.env.PORT || 5000;
  server.listen(PORT, "0.0.0.0", () => {
    log(`serving on port ${PORT}`);
  });
})();
```

#server/routes.ts
```typescript
import type { Express } from "express";
import { createServer, type Server } from "http";
import jwt from "jsonwebtoken";
import { storage } from "./storage";
import { insertLeadSchema, loginSchema, updateLeadStatusSchema } from "@shared/schema";
import { predictLeadQuality } from "../client/src/lib/ai-model";

const JWT_SECRET = process.env.JWT_SECRET || "your-super-secret-jwt-key";

// JWT Authentication middleware
const authenticateToken = (req: any, res: any, next: any) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }

  jwt.verify(token, JWT_SECRET, (err: any, user: any) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

// Mock CRM integration
const syncToCRM = async (lead: any) => {
  await new Promise(resolve => setTimeout(resolve, 500));
  return {
    crmId: `SF_${Date.now()}`,
    synced: true,
    syncedAt: new Date(),
  };
};

export async function registerRoutes(app: Express): Promise<Server> {
  
  // Authentication routes
  app.post("/api/auth/login", async (req, res) => {
    try {
      const { username, password } = loginSchema.parse(req.body);
      
      const user = await storage.verifyUser(username, password);
      if (!user) {
        return res.status(401).json({ message: "Invalid credentials" });
      }

      const token = jwt.sign(
        { id: user.id, username: user.username, role: user.role },
        JWT_SECRET,
        { expiresIn: '24h' }
      );

      res.json({
        token,
        user: {
          id: user.id,
          username: user.username,
          role: user.role,
        },
      });
    } catch (error) {
      res.status(400).json({ message: "Invalid request data" });
    }
  });

  app.post("/api/auth/verify", authenticateToken, async (req, res) => {
    res.json({ user: req.user });
  });

  // Lead management routes
  app.get("/api/leads", authenticateToken, async (req, res) => {
    try {
      const { quality, status, search } = req.query;
      
      let leads;
      if (search) {
        leads = await storage.searchLeads(search as string);
      } else if (quality) {
        leads = await storage.getLeadsByQuality(quality as string);
      } else if (status) {
        leads = await storage.getLeadsByStatus(status as string);
      } else {
        leads = await storage.getLeads();
      }
      
      res.json(leads);
    } catch (error) {
      res.status(500).json({ message: "Failed to fetch leads" });
    }
  });

  app.post("/api/leads", async (req, res) => {
    try {
      const leadData = insertLeadSchema.parse(req.body);
      
      // Apply AI scoring
      const aiResult = await predictLeadQuality(leadData);
      const leadWithAI = {
        ...leadData,
        score: aiResult.score,
        quality: aiResult.quality,
        aiPrediction: aiResult.prediction,
      };
      
      const lead = await storage.createLead(leadWithAI);
      
      // Auto-sync high quality leads to CRM
      if (lead.quality === "high") {
        try {
          const crmResult = await syncToCRM(lead);
          lead.crmSynced = crmResult.synced;
          lead.crmId = crmResult.crmId;
        } catch (error) {
          console.error("CRM sync failed:", error);
        }
      }
      
      res.status(201).json(lead);
    } catch (error) {
      console.error("Create lead error:", error);
      res.status(400).json({ message: "Invalid lead data" });
    }
  });

  app.patch("/api/leads/:id/status", authenticateToken, async (req, res) => {
    try {
      const id = parseInt(req.params.id);
      const statusUpdate = updateLeadStatusSchema.parse(req.body);
      
      const updatedLead = await storage.updateLeadStatus(id, statusUpdate);
      if (!updatedLead) {
        return res.status(404).json({ message: "Lead not found" });
      }
      
      res.json(updatedLead);
    } catch (error) {
      res.status(400).json({ message: "Invalid status update" });
    }
  });

  // Analytics routes
  app.get("/api/analytics/stats", authenticateToken, async (req, res) => {
    try {
      const stats = await storage.getLeadStats();
      res.json(stats);
    } catch (error) {
      res.status(500).json({ message: "Failed to fetch analytics" });
    }
  });

  // Chatbot routes
  app.post("/api/chat/message", async (req, res) => {
    try {
      const { sessionId, message, leadData } = req.body;
      
      let session = await storage.getChatSession(sessionId);
      if (!session) {
        session = await storage.createChatSession({
          sessionId,
          messages: [],
          leadData: null,
        });
      }
      
      const messages = Array.isArray(session.messages) ? session.messages : [];
      
      // AI-powered responses
      let botResponse = "I'd be happy to help you with information about our programs. What specific field are you interested in?";
      
      const lowerMessage = message.toLowerCase();
      if (lowerMessage.includes("computer science") || lowerMessage.includes("programming")) {
        botResponse = "Excellent choice! Our Computer Science program covers AI, machine learning, software development, and more. What's your current background?";
      } else if (lowerMessage.includes("data science") || lowerMessage.includes("analytics")) {
        botResponse = "Data Science is in high demand! Our program includes Python, R, machine learning, and business analytics. Are you looking to transition careers?";
      } else if (lowerMessage.includes("cybersecurity") || lowerMessage.includes("security")) {
        botResponse = "Cybersecurity is critical today! Our program covers ethical hacking, network security, and compliance. Do you have any IT background?";
      } else if (lowerMessage.includes("cost") || lowerMessage.includes("price") || lowerMessage.includes("tuition")) {
        botResponse = "I can connect you with our admissions team for detailed pricing information. May I get your contact information to have someone reach out?";
      } else if (lowerMessage.includes("hello") || lowerMessage.includes("hi")) {
        botResponse = "Hello! Welcome to EduLead Pro. I'm here to help you explore our educational programs. What field interests you most?";
      }
      
      const updatedMessages = [
        ...messages,
        { role: "user", content: message, timestamp: new Date() },
        { role: "bot", content: botResponse, timestamp: new Date() },
      ];
      
      await storage.updateChatSession(sessionId, updatedMessages, leadData);
      
      res.json({ response: botResponse, sessionId });
    } catch (error) {
      console.error("Chat error:", error);
      res.status(500).json({ message: "Chat service unavailable" });
    }
  });

  // CRM sync route
  app.post("/api/crm/sync/:leadId", authenticateToken, async (req, res) => {
    try {
      const leadId = parseInt(req.params.leadId);
      const lead = await storage.getLead(leadId);
      
      if (!lead) {
        return res.status(404).json({ message: "Lead not found" });
      }
      
      const crmResult = await syncToCRM(lead);
      
      res.json({
        message: "Lead synced to CRM successfully",
        crmId: crmResult.crmId,
        syncedAt: crmResult.syncedAt,
      });
    } catch (error) {
      res.status(500).json({ message: "CRM sync failed" });
    }
  });

  const httpServer = createServer(app);
  return httpServer;
}
```

#server/vite.ts
```typescript
import { Express } from "express";
import { createServer as createViteServer, type ViteDevServer } from "vite";
import { Server } from "http";
import fs from "fs";
import path from "path";

export function log(message: string, source = "express") {
  const timestamp = new Date().toLocaleTimeString("en-US", {
    hour12: false,
  });
  console.log(`${timestamp} [${source}] ${message}`);
}

export async function setupVite(app: Express, server: Server) {
  const vite = await createViteServer({
    server: { middlewareMode: true },
    appType: "spa",
  });

  app.use(vite.ssrFixStacktrace);
  app.use(vite.middlewares);
}

export function serveStatic(app: Express) {
  const distPath = path.resolve("dist/public");
  
  if (!fs.existsSync(distPath)) {
    throw new Error(
      `Could not find the frontend build! You need to run "npm run build" to create the production build.`
    );
  }

  app.use(express.static(distPath));
  app.get("*", (_req, res) => {
    res.sendFile(path.resolve(distPath, "index.html"));
  });
}
```

#client/src/lib/ai-model.ts
```typescript
import * as tf from '@tensorflow/tfjs';

interface LeadData {
  age?: number;
  location?: string;
  interest: string;
  degreeLevel?: string;
  timeline?: string;
  comments?: string;
}

interface PredictionResult {
  score: number;
  quality: "high" | "medium" | "low";
  prediction: {
    confidence: number;
    factors: string[];
  };
}

// Education sector mapping values
const interestMapping: Record<string, number> = {
  'computer-science': 0.9,
  'data-science': 0.85,
  'cybersecurity': 0.9,
  'software-engineering': 0.85,
  'ai-machine-learning': 0.95,
  'business-analytics': 0.7,
  'business': 0.6,
  'healthcare': 0.75,
  'engineering': 0.8,
  'education': 0.5,
  'arts': 0.3,
  'other': 0.4,
};

const degreeLevelMapping: Record<string, number> = {
  'doctoral': 0.95,
  'masters': 0.85,
  'bachelors': 0.7,
  'associates': 0.5,
  'certificate': 0.4,
};

const timelineMapping: Record<string, number> = {
  'immediately': 0.95,
  '1-3months': 0.85,
  '3-6months': 0.7,
  '6-12months': 0.5,
  'undecided': 0.3,
};

let model: tf.LayersModel | null = null;

// Create TensorFlow.js neural network model
async function createModel(): Promise<tf.LayersModel> {
  if (model) return model;

  model = tf.sequential({
    layers: [
      tf.layers.dense({ units: 16, activation: 'relu', inputShape: [6] }),
      tf.layers.dropout({ rate: 0.2 }),
      tf.layers.dense({ units: 8, activation: 'relu' }),
      tf.layers.dense({ units: 1, activation: 'sigmoid' })
    ]
  });

  model.compile({
    optimizer: tf.train.adam(0.001),
    loss: 'binaryCrossentropy',
    metrics: ['accuracy']
  });

  await trainModel(model);
  return model;
}

// Train the neural network with education sector data
async function trainModel(model: tf.LayersModel): Promise<void> {
  const trainingData = generateTrainingData();
  
  const xs = tf.tensor2d(trainingData.features);
  const ys = tf.tensor2d(trainingData.labels, [trainingData.labels.length, 1]);

  await model.fit(xs, ys, {
    epochs: 50,
    batchSize: 32,
    verbose: 0,
    validationSplit: 0.2,
  });

  xs.dispose();
  ys.dispose();
}

// Generate synthetic training data for education leads
function generateTrainingData(): { features: number[][], labels: number[] } {
  const features: number[][] = [];
  const labels: number[] = [];

  for (let i = 0; i < 1000; i++) {
    const age = Math.random() * 50 + 18;
    const interest = Object.keys(interestMapping)[Math.floor(Math.random() * Object.keys(interestMapping).length)];
    const degreeLevel = Object.keys(degreeLevelMapping)[Math.floor(Math.random() * Object.keys(degreeLevelMapping).length)];
    const timeline = Object.keys(timelineMapping)[Math.floor(Math.random() * Object.keys(timelineMapping).length)];
    
    const feature = [
      age / 100,
      interestMapping[interest] || 0.5,
      degreeLevelMapping[degreeLevel] || 0.5,
      timelineMapping[timeline] || 0.5,
      Math.random() * 0.5 + 0.25,
      Math.random() * 0.5 + 0.25,
    ];

    const score = (
      feature[1] * 0.3 +
      feature[2] * 0.25 +
      feature[3] * 0.25 +
      feature[4] * 0.1 +
      feature[5] * 0.1
    );

    features.push(feature);
    labels.push(score > 0.7 ? 1 : 0);
  }

  return { features, labels };
}

// Extract features from lead data for AI model
function extractFeatures(leadData: LeadData): number[] {
  const age = leadData.age || 25;
  const interest = leadData.interest?.toLowerCase().replace(/\s+/g, '-') || 'other';
  const degreeLevel = leadData.degreeLevel || 'bachelors';
  const timeline = leadData.timeline || 'undecided';
  
  const commentScore = leadData.comments 
    ? (leadData.comments.toLowerCase().includes('career') ? 0.8 :
       leadData.comments.toLowerCase().includes('interested') ? 0.7 : 0.5)
    : 0.5;

  return [
    age / 100,
    interestMapping[interest] || interestMapping['other'],
    degreeLevelMapping[degreeLevel] || degreeLevelMapping['bachelors'],
    timelineMapping[timeline] || timelineMapping['undecided'],
    Math.random() * 0.5 + 0.25,
    commentScore,
  ];
}

// Analyze factors contributing to lead quality
function analyzeFactors(leadData: LeadData, score: number): string[] {
  const factors: string[] = [];
  
  const interest = leadData.interest?.toLowerCase().replace(/\s+/g, '-') || 'other';
  if ((interestMapping[interest] || 0) > 0.8) {
    factors.push('high_demand_field');
  }
  
  if (leadData.timeline === 'immediately' || leadData.timeline === '1-3months') {
    factors.push('immediate_timeline');
  }
  
  if (leadData.degreeLevel === 'masters' || leadData.degreeLevel === 'doctoral') {
    factors.push('advanced_degree');
  }
  
  if (leadData.comments && leadData.comments.toLowerCase().includes('career')) {
    factors.push('career_focused');
  }
  
  if (score > 85) {
    factors.push('high_conversion_probability');
  }
  
  if (factors.length === 0) {
    factors.push('standard_profile');
  }
  
  return factors;
}

// Main prediction function using TensorFlow.js model
export async function predictLeadQuality(leadData: LeadData): Promise<PredictionResult> {
  try {
    const model = await createModel();
    const features = extractFeatures(leadData);
    
    const prediction = model.predict(tf.tensor2d([features])) as tf.Tensor;
    const predictionValue = await prediction.data();
    prediction.dispose();
    
    const confidence = predictionValue[0];
    const score = Math.round(confidence * 100);
    
    let quality: "high" | "medium" | "low";
    if (score >= 80) {
      quality = "high";
    } else if (score >= 60) {
      quality = "medium";
    } else {
      quality = "low";
    }
    
    const factors = analyzeFactors(leadData, score);
    
    return {
      score,
      quality,
      prediction: {
        confidence,
        factors,
      },
    };
  } catch (error) {
    console.error('AI prediction error:', error);
    
    // Fallback scoring if AI model fails
    const interest = leadData.interest?.toLowerCase().replace(/\s+/g, '-') || 'other';
    const interestScore = (interestMapping[interest] || 0.5) * 100;
    const timelineScore = (timelineMapping[leadData.timeline || 'undecided'] || 0.5) * 100;
    const degreeScore = (degreeLevelMapping[leadData.degreeLevel || 'bachelors'] || 0.5) * 100;
    
    const fallbackScore = Math.round((interestScore + timelineScore + degreeScore) / 3);
    
    return {
      score: fallbackScore,
      quality: fallbackScore >= 80 ? "high" : fallbackScore >= 60 ? "medium" : "low",
      prediction: {
        confidence: fallbackScore / 100,
        factors: ["rule_based_scoring"],
      },
    };
  }
}

// Initialize TensorFlow.js when module loads
tf.ready().then(() => {
  console.log('TensorFlow.js is ready!');
});
```

#tsconfig.json
```json
{
  "include": ["client/src/**/*", "shared/**/*", "server/**/*"],
  "exclude": ["node_modules", "build", "dist", "**/*.test.ts"],
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": "./node_modules/typescript/tsbuildinfo",
    "noEmit": true,
    "module": "ESNext",
    "strict": true,
    "lib": ["esnext", "dom", "dom.iterable"],
    "jsx": "preserve",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "allowImportingTsExtensions": true,
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "types": ["node", "vite/client"],
    "paths": {
      "@/*": ["./client/src/*"],
      "@shared/*": ["./shared/*"]
    }
  }
}
```

#vite.config.ts
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./client/src"),
      "@shared": path.resolve(__dirname, "./shared"),
    },
  },
  root: path.resolve(__dirname, "client"),
  build: {
    outDir: path.resolve(__dirname, "dist/public"),
    emptyOutDir: true,
  },
});
```

#tailwind.config.ts
```typescript
import type { Config } from "tailwindcss";

export default {
  darkMode: ["class"],
  content: [
    "./client/src/**/*.{js,ts,jsx,tsx}",
    "./client/index.html"
  ],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
} satisfies Config;
```
