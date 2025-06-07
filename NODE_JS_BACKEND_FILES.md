#server/storage.ts
```typescript
import { leads, users, chatSessions, type Lead, type InsertLead, type User, type InsertUser, type ChatSession, type InsertChatSession, type UpdateLeadStatus } from "@shared/schema";
import bcrypt from "bcryptjs";

export interface IStorage {
  // User management
  getUser(id: number): Promise<User | undefined>;
  getUserByUsername(username: string): Promise<User | undefined>;
  createUser(user: InsertUser): Promise<User>;
  verifyUser(username: string, password: string): Promise<User | null>;

  // Lead management
  getLeads(): Promise<Lead[]>;
  getLead(id: number): Promise<Lead | undefined>;
  createLead(lead: InsertLead): Promise<Lead>;
  updateLeadStatus(id: number, status: UpdateLeadStatus): Promise<Lead | undefined>;
  getLeadsByQuality(quality: string): Promise<Lead[]>;
  getLeadsByStatus(status: string): Promise<Lead[]>;
  searchLeads(query: string): Promise<Lead[]>;

  // Chat sessions
  getChatSession(sessionId: string): Promise<ChatSession | undefined>;
  createChatSession(session: InsertChatSession): Promise<ChatSession>;
  updateChatSession(sessionId: string, messages: any[], leadData?: any): Promise<ChatSession | undefined>;

  // Analytics
  getLeadStats(): Promise<{
    totalLeads: number;
    highQuality: number;
    mediumQuality: number;
    lowQuality: number;
    conversionRate: number;
    newLeads: number;
    contactedLeads: number;
    qualifiedLeads: number;
    convertedLeads: number;
  }>;
}

export class MemStorage implements IStorage {
  private users: Map<number, User>;
  private leads: Map<number, Lead>;
  private chatSessions: Map<string, ChatSession>;
  private currentUserId: number;
  private currentLeadId: number;
  private currentChatId: number;

  constructor() {
    this.users = new Map();
    this.leads = new Map();
    this.chatSessions = new Map();
    this.currentUserId = 1;
    this.currentLeadId = 1;
    this.currentChatId = 1;

    this.initializeDefaultUser();
    this.initializeSampleLeads();
  }

  private async initializeDefaultUser() {
    const hashedPassword = await bcrypt.hash("admin123", 10);
    const adminUser: User = {
      id: this.currentUserId++,
      username: "admin",
      password: hashedPassword,
      role: "admin",
      createdAt: new Date(),
    };
    this.users.set(adminUser.id, adminUser);
  }

  private initializeSampleLeads() {
    const sampleLeads: Lead[] = [
      {
        id: this.currentLeadId++,
        name: "Sarah Johnson",
        email: "sarah.johnson@email.com",
        phone: "(555) 123-4567",
        age: 24,
        location: "California, USA",
        interest: "Computer Science",
        degreeLevel: "bachelors",
        timeline: "1-3months",
        comments: "Interested in AI/ML specialization, currently working as a junior developer",
        score: 92,
        quality: "high",
        status: "qualified",
        aiPrediction: { 
          confidence: 0.92, 
          factors: ["high_demand_field", "immediate_timeline", "clear_goals", "relevant_background"] 
        },
        crmSynced: true,
        crmId: "SF_001_2024",
        createdAt: new Date("2024-01-15"),
        updatedAt: new Date("2024-01-15"),
      },
      {
        id: this.currentLeadId++,
        name: "Michael Chen",
        email: "michael.chen@email.com",
        phone: "(555) 234-5678",
        age: 28,
        location: "New York, USA",
        interest: "Data Science",
        degreeLevel: "masters",
        timeline: "3-6months",
        comments: "Career transition from finance to data science, have basic Python knowledge",
        score: 87,
        quality: "high",
        status: "contacted",
        aiPrediction: { 
          confidence: 0.87, 
          factors: ["career_transition", "masters_level", "relevant_background", "high_demand_field"] 
        },
        crmSynced: true,
        crmId: "SF_002_2024",
        createdAt: new Date("2024-01-14"),
        updatedAt: new Date("2024-01-14"),
      },
      {
        id: this.currentLeadId++,
        name: "Emily Rodriguez",
        email: "emily.rodriguez@email.com",
        phone: "(555) 345-6789",
        age: 22,
        location: "Texas, USA",
        interest: "Business Analytics",
        degreeLevel: "bachelors",
        timeline: "6-12months",
        comments: "Recent graduate looking to upskill in analytics and data visualization",
        score: 75,
        quality: "medium",
        status: "new",
        aiPrediction: { 
          confidence: 0.75, 
          factors: ["recent_graduate", "business_background", "delayed_timeline"] 
        },
        crmSynced: false,
        crmId: null,
        createdAt: new Date("2024-01-13"),
        updatedAt: new Date("2024-01-13"),
      },
      {
        id: this.currentLeadId++,
        name: "David Thompson",
        email: "david.thompson@email.com",
        phone: "(555) 456-7890",
        age: 35,
        location: "Florida, USA",
        interest: "Cybersecurity",
        degreeLevel: "masters",
        timeline: "immediately",
        comments: "Military background with security clearance, looking for advanced cybersecurity certification",
        score: 96,
        quality: "high",
        status: "converted",
        aiPrediction: { 
          confidence: 0.96, 
          factors: ["military_background", "security_clearance", "immediate_start", "high_demand_field", "advanced_degree"] 
        },
        crmSynced: true,
        crmId: "SF_003_2024",
        createdAt: new Date("2024-01-12"),
        updatedAt: new Date("2024-01-12"),
      },
      {
        id: this.currentLeadId++,
        name: "Lisa Wang",
        email: "lisa.wang@email.com",
        phone: "(555) 567-8901",
        age: 26,
        location: "Washington, USA",
        interest: "AI/Machine Learning",
        degreeLevel: "masters",
        timeline: "1-3months",
        comments: "PhD in Mathematics, want to transition to practical AI applications in industry",
        score: 94,
        quality: "high",
        status: "qualified",
        aiPrediction: { 
          confidence: 0.94, 
          factors: ["advanced_degree", "high_demand_field", "immediate_timeline", "strong_background"] 
        },
        crmSynced: true,
        crmId: "SF_004_2024",
        createdAt: new Date("2024-01-11"),
        updatedAt: new Date("2024-01-11"),
      },
      {
        id: this.currentLeadId++,
        name: "James Wilson",
        email: "james.wilson@email.com",
        phone: "(555) 678-9012",
        age: 30,
        location: "Illinois, USA",
        interest: "Software Engineering",
        degreeLevel: "bachelors",
        timeline: "3-6months",
        comments: "Full-stack developer wanting to specialize in cloud architecture and DevOps",
        score: 83,
        quality: "high",
        status: "contacted",
        aiPrediction: { 
          confidence: 0.83, 
          factors: ["relevant_background", "specialization_focus", "high_demand_field"] 
        },
        crmSynced: false,
        crmId: null,
        createdAt: new Date("2024-01-10"),
        updatedAt: new Date("2024-01-10"),
      },
      {
        id: this.currentLeadId++,
        name: "Maria Garcia",
        email: "maria.garcia@email.com",
        phone: "(555) 789-0123",
        age: 33,
        location: "California, USA",
        interest: "Healthcare",
        degreeLevel: "masters",
        timeline: "6-12months",
        comments: "Nurse practitioner interested in healthcare informatics and digital health solutions",
        score: 78,
        quality: "medium",
        status: "new",
        aiPrediction: { 
          confidence: 0.78, 
          factors: ["healthcare_background", "career_enhancement", "delayed_timeline"] 
        },
        crmSynced: false,
        crmId: null,
        createdAt: new Date("2024-01-09"),
        updatedAt: new Date("2024-01-09"),
      },
      {
        id: this.currentLeadId++,
        name: "Robert Kim",
        email: "robert.kim@email.com",
        phone: "(555) 890-1234",
        age: 29,
        location: "Oregon, USA",
        interest: "Business",
        degreeLevel: "masters",
        timeline: "immediately",
        comments: "MBA graduate seeking executive leadership program with focus on digital transformation",
        score: 81,
        quality: "high",
        status: "qualified",
        aiPrediction: { 
          confidence: 0.81, 
          factors: ["mba_background", "immediate_start", "leadership_focus"] 
        },
        crmSynced: false,
        crmId: null,
        createdAt: new Date("2024-01-08"),
        updatedAt: new Date("2024-01-08"),
      }
    ];

    sampleLeads.forEach(lead => {
      this.leads.set(lead.id, lead);
    });
  }

  async getUser(id: number): Promise<User | undefined> {
    return this.users.get(id);
  }

  async getUserByUsername(username: string): Promise<User | undefined> {
    return Array.from(this.users.values()).find(user => user.username === username);
  }

  async createUser(insertUser: InsertUser): Promise<User> {
    const hashedPassword = await bcrypt.hash(insertUser.password, 10);
    const user: User = {
      ...insertUser,
      id: this.currentUserId++,
      password: hashedPassword,
      createdAt: new Date(),
    };
    this.users.set(user.id, user);
    return user;
  }

  async verifyUser(username: string, password: string): Promise<User | null> {
    const user = await this.getUserByUsername(username);
    if (!user) return null;
    
    const isValid = await bcrypt.compare(password, user.password);
    return isValid ? user : null;
  }

  async getLeads(): Promise<Lead[]> {
    return Array.from(this.leads.values()).sort((a, b) => 
      new Date(b.createdAt!).getTime() - new Date(a.createdAt!).getTime()
    );
  }

  async getLead(id: number): Promise<Lead | undefined> {
    return this.leads.get(id);
  }

  async createLead(insertLead: InsertLead): Promise<Lead> {
    const lead: Lead = {
      ...insertLead,
      id: this.currentLeadId++,
      createdAt: new Date(),
      updatedAt: new Date(),
      crmSynced: false,
      crmId: null,
    };
    this.leads.set(lead.id, lead);
    return lead;
  }

  async updateLeadStatus(id: number, statusUpdate: UpdateLeadStatus): Promise<Lead | undefined> {
    const lead = this.leads.get(id);
    if (!lead) return undefined;

    const updatedLead: Lead = {
      ...lead,
      status: statusUpdate.status,
      updatedAt: new Date(),
    };
    this.leads.set(id, updatedLead);
    return updatedLead;
  }

  async getLeadsByQuality(quality: string): Promise<Lead[]> {
    return Array.from(this.leads.values()).filter(lead => lead.quality === quality);
  }

  async getLeadsByStatus(status: string): Promise<Lead[]> {
    return Array.from(this.leads.values()).filter(lead => lead.status === status);
  }

  async searchLeads(query: string): Promise<Lead[]> {
    const lowercaseQuery = query.toLowerCase();
    return Array.from(this.leads.values()).filter(lead => 
      lead.name.toLowerCase().includes(lowercaseQuery) ||
      lead.email.toLowerCase().includes(lowercaseQuery) ||
      lead.interest.toLowerCase().includes(lowercaseQuery) ||
      (lead.location && lead.location.toLowerCase().includes(lowercaseQuery)) ||
      (lead.comments && lead.comments.toLowerCase().includes(lowercaseQuery))
    );
  }

  async getChatSession(sessionId: string): Promise<ChatSession | undefined> {
    return this.chatSessions.get(sessionId);
  }

  async createChatSession(insertSession: InsertChatSession): Promise<ChatSession> {
    const session: ChatSession = {
      ...insertSession,
      id: this.currentChatId++,
      createdAt: new Date(),
    };
    this.chatSessions.set(session.sessionId, session);
    return session;
  }

  async updateChatSession(sessionId: string, messages: any[], leadData?: any): Promise<ChatSession | undefined> {
    const session = this.chatSessions.get(sessionId);
    if (!session) return undefined;

    const updatedSession: ChatSession = {
      ...session,
      messages,
      leadData: leadData || session.leadData,
    };
    this.chatSessions.set(sessionId, updatedSession);
    return updatedSession;
  }

  async getLeadStats() {
    const allLeads = Array.from(this.leads.values());
    
    return {
      totalLeads: allLeads.length,
      highQuality: allLeads.filter(l => l.quality === "high").length,
      mediumQuality: allLeads.filter(l => l.quality === "medium").length,
      lowQuality: allLeads.filter(l => l.quality === "low").length,
      conversionRate: Math.round((allLeads.filter(l => l.status === "converted").length / allLeads.length) * 100 * 10) / 10,
      newLeads: allLeads.filter(l => l.status === "new").length,
      contactedLeads: allLeads.filter(l => l.status === "contacted").length,
      qualifiedLeads: allLeads.filter(l => l.status === "qualified").length,
      convertedLeads: allLeads.filter(l => l.status === "converted").length,
    };
  }
}

export const storage = new MemStorage();
```
