#client/src/App.tsx
```tsx
import { Route, Switch, useLocation } from "wouter";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { AuthProvider, useAuth } from "@/hooks/use-auth";
import { Toaster } from "@/components/ui/toaster";
import AdminLogin from "@/pages/admin-login";
import Dashboard from "@/pages/dashboard";
import LeadForm from "@/pages/lead-form";
import Leads from "@/pages/leads";
import NotFound from "@/pages/not-found";
import { queryClient } from "@/lib/queryClient";

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  const [, setLocation] = useLocation();

  if (!isAuthenticated) {
    setLocation("/admin-login");
    return null;
  }

  return <>{children}</>;
}

function PublicRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  const [, setLocation] = useLocation();

  if (isAuthenticated) {
    setLocation("/dashboard");
    return null;
  }

  return <>{children}</>;
}

function Router() {
  return (
    <Switch>
      <Route path="/" component={() => <LeadForm />} />
      <Route path="/lead-form" component={() => <LeadForm />} />
      <Route path="/admin-login">
        <PublicRoute>
          <AdminLogin />
        </PublicRoute>
      </Route>
      <Route path="/dashboard">
        <ProtectedRoute>
          <Dashboard />
        </ProtectedRoute>
      </Route>
      <Route path="/leads">
        <ProtectedRoute>
          <Leads />
        </ProtectedRoute>
      </Route>
      <Route component={NotFound} />
    </Switch>
  );
}

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <div className="min-h-screen bg-background">
          <Router />
          <Toaster />
        </div>
      </AuthProvider>
    </QueryClientProvider>
  );
}

export default App;
```

#client/src/main.tsx
```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.tsx";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

#client/src/pages/lead-form.tsx
```tsx
import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useMutation } from "@tanstack/react-query";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Progress } from "@/components/ui/progress";
import { useToast } from "@/hooks/use-toast";
import { apiRequest } from "@/lib/queryClient";
import { AIScoring } from "@/components/ai-scoring";
import { Chatbot } from "@/components/chatbot";
import { insertLeadSchema } from "@shared/schema";

const publicLeadSchema = insertLeadSchema.extend({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Please enter a valid email address"),
});

type PublicLeadForm = z.infer<typeof publicLeadSchema>;

export default function LeadForm() {
  const [aiResult, setAiResult] = useState<any>(null);
  const [isSubmitted, setIsSubmitted] = useState(false);
  const { toast } = useToast();

  const {
    register,
    handleSubmit,
    formState: { errors },
    setValue,
    watch,
  } = useForm<PublicLeadForm>({
    resolver: zodResolver(publicLeadSchema),
    defaultValues: {
      interest: "",
      degreeLevel: "",
      timeline: "",
    },
  });

  const mutation = useMutation({
    mutationFn: async (data: PublicLeadForm) => {
      const response = await apiRequest("/api/leads", {
        method: "POST",
        body: JSON.stringify(data),
      });
      return response;
    },
    onSuccess: (data) => {
      setAiResult(data);
      setIsSubmitted(true);
      toast({
        title: "Application Submitted Successfully!",
        description: `AI Quality Score: ${data.score}% (${data.quality} priority)`,
      });
    },
    onError: (error) => {
      toast({
        title: "Submission Failed",
        description: "Please check your information and try again.",
        variant: "destructive",
      });
    },
  });

  const onSubmit = (data: PublicLeadForm) => {
    mutation.mutate(data);
  };

  if (isSubmitted && aiResult) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
        <Card className="w-full max-w-2xl">
          <CardHeader className="text-center">
            <CardTitle className="text-3xl font-bold text-green-600">
              Thank You for Your Interest!
            </CardTitle>
            <CardDescription className="text-lg mt-2">
              Your application has been submitted and processed by our AI system
            </CardDescription>
          </CardHeader>
          <CardContent className="space-y-6">
            <AIScoring
              score={aiResult.score}
              quality={aiResult.quality}
              prediction={aiResult.aiPrediction}
              className="mb-6"
            />
            
            <div className="bg-blue-50 p-4 rounded-lg">
              <h3 className="font-semibold text-blue-900 mb-2">What happens next?</h3>
              <ul className="text-blue-800 space-y-1">
                <li>• Our admissions team will review your application</li>
                <li>• You'll receive a follow-up email within 24-48 hours</li>
                <li>• High-priority applications are fast-tracked to our advisors</li>
                <li>• We'll schedule a consultation to discuss your educational goals</li>
              </ul>
            </div>

            {aiResult.quality === "high" && (
              <div className="bg-green-50 border border-green-200 p-4 rounded-lg">
                <div className="flex items-center">
                  <Badge variant="secondary" className="bg-green-100 text-green-800">
                    Priority Application
                  </Badge>
                </div>
                <p className="text-green-800 mt-2">
                  Your application has been automatically forwarded to our senior admissions team for expedited processing.
                </p>
              </div>
            )}

            <Button 
              onClick={() => window.location.reload()} 
              className="w-full"
              variant="outline"
            >
              Submit Another Application
            </Button>
          </CardContent>
        </Card>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
      {/* Header */}
      <div className="bg-white shadow-sm border-b">
        <div className="max-w-7xl mx-auto px-4 py-6">
          <div className="text-center">
            <h1 className="text-4xl font-bold text-gray-900">EduLead Pro</h1>
            <p className="text-xl text-gray-600 mt-2">AI-Powered Education Matching</p>
          </div>
        </div>
      </div>

      <div className="max-w-4xl mx-auto p-6 grid grid-cols-1 lg:grid-cols-3 gap-8">
        {/* Main Form */}
        <div className="lg:col-span-2">
          <Card>
            <CardHeader>
              <CardTitle className="text-2xl">Apply for Educational Programs</CardTitle>
              <CardDescription>
                Tell us about your educational goals and let our AI help match you with the perfect program
              </CardDescription>
            </CardHeader>
            <CardContent>
              <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
                {/* Personal Information */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div>
                    <Label htmlFor="name">Full Name *</Label>
                    <Input
                      id="name"
                      {...register("name")}
                      placeholder="Enter your full name"
                      className={errors.name ? "border-red-500" : ""}
                    />
                    {errors.name && (
                      <p className="text-red-500 text-sm mt-1">{errors.name.message}</p>
                    )}
                  </div>

                  <div>
                    <Label htmlFor="email">Email Address *</Label>
                    <Input
                      id="email"
                      type="email"
                      {...register("email")}
                      placeholder="your.email@example.com"
                      className={errors.email ? "border-red-500" : ""}
                    />
                    {errors.email && (
                      <p className="text-red-500 text-sm mt-1">{errors.email.message}</p>
                    )}
                  </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div>
                    <Label htmlFor="phone">Phone Number</Label>
                    <Input
                      id="phone"
                      {...register("phone")}
                      placeholder="(555) 123-4567"
                    />
                  </div>

                  <div>
                    <Label htmlFor="age">Age</Label>
                    <Input
                      id="age"
                      type="number"
                      {...register("age", { valueAsNumber: true })}
                      placeholder="25"
                      min="16"
                      max="100"
                    />
                  </div>
                </div>

                <div>
                  <Label htmlFor="location">Location</Label>
                  <Input
                    id="location"
                    {...register("location")}
                    placeholder="City, State"
                  />
                </div>

                {/* Educational Preferences */}
                <div className="border-t pt-6">
                  <h3 className="text-lg font-semibold mb-4">Educational Preferences</h3>
                  
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div>
                      <Label htmlFor="interest">Field of Interest *</Label>
                      <Select onValueChange={(value) => setValue("interest", value)}>
                        <SelectTrigger className={errors.interest ? "border-red-500" : ""}>
                          <SelectValue placeholder="Select your field of interest" />
                        </SelectTrigger>
                        <SelectContent>
                          <SelectItem value="computer-science">Computer Science</SelectItem>
                          <SelectItem value="data-science">Data Science</SelectItem>
                          <SelectItem value="cybersecurity">Cybersecurity</SelectItem>
                          <SelectItem value="software-engineering">Software Engineering</SelectItem>
                          <SelectItem value="ai-machine-learning">AI/Machine Learning</SelectItem>
                          <SelectItem value="business-analytics">Business Analytics</SelectItem>
                          <SelectItem value="business">Business Administration</SelectItem>
                          <SelectItem value="healthcare">Healthcare</SelectItem>
                          <SelectItem value="engineering">Engineering</SelectItem>
                          <SelectItem value="education">Education</SelectItem>
                          <SelectItem value="arts">Arts & Creative</SelectItem>
                          <SelectItem value="other">Other</SelectItem>
                        </SelectContent>
                      </Select>
                      {errors.interest && (
                        <p className="text-red-500 text-sm mt-1">{errors.interest.message}</p>
                      )}
                    </div>

                    <div>
                      <Label htmlFor="degreeLevel">Desired Degree Level</Label>
                      <Select onValueChange={(value) => setValue("degreeLevel", value)}>
                        <SelectTrigger>
                          <SelectValue placeholder="Select degree level" />
                        </SelectTrigger>
                        <SelectContent>
                          <SelectItem value="certificate">Certificate Program</SelectItem>
                          <SelectItem value="associates">Associate's Degree</SelectItem>
                          <SelectItem value="bachelors">Bachelor's Degree</SelectItem>
                          <SelectItem value="masters">Master's Degree</SelectItem>
                          <SelectItem value="doctoral">Doctoral Degree</SelectItem>
                        </SelectContent>
                      </Select>
                    </div>
                  </div>

                  <div className="mt-4">
                    <Label htmlFor="timeline">When would you like to start?</Label>
                    <Select onValueChange={(value) => setValue("timeline", value)}>
                      <SelectTrigger>
                        <SelectValue placeholder="Select your preferred timeline" />
                      </SelectTrigger>
                      <SelectContent>
                        <SelectItem value="immediately">Immediately</SelectItem>
                        <SelectItem value="1-3months">1-3 months</SelectItem>
                        <SelectItem value="3-6months">3-6 months</SelectItem>
                        <SelectItem value="6-12months">6-12 months</SelectItem>
                        <SelectItem value="undecided">I'm not sure yet</SelectItem>
                      </SelectContent>
                    </Select>
                  </div>
                </div>

                {/* Additional Information */}
                <div className="border-t pt-6">
                  <Label htmlFor="comments">Additional Information</Label>
                  <Textarea
                    id="comments"
                    {...register("comments")}
                    placeholder="Tell us about your career goals, previous experience, or any specific questions you have..."
                    rows={4}
                  />
                </div>

                <Button 
                  type="submit" 
                  className="w-full" 
                  size="lg"
                  disabled={mutation.isPending}
                >
                  {mutation.isPending ? (
                    <>
                      <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-white mr-2"></div>
                      Processing with AI...
                    </>
                  ) : (
                    "Submit Application"
                  )}
                </Button>
              </form>
            </CardContent>
          </Card>
        </div>

        {/* Sidebar */}
        <div className="space-y-6">
          {/* Chat Assistant */}
          <Chatbot />

          {/* Info Cards */}
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">AI-Powered Matching</CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-sm text-gray-600 mb-3">
                Our advanced AI system analyzes your profile to:
              </p>
              <ul className="text-sm space-y-1">
                <li>• Match you with optimal programs</li>
                <li>• Predict your success probability</li>
                <li>• Prioritize high-potential applications</li>
                <li>• Connect you with the right advisors</li>
              </ul>
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Why Choose EduLead Pro?</CardTitle>
            </CardHeader>
            <CardContent>
              <ul className="text-sm space-y-2">
                <li>• 90%+ job placement rate</li>
                <li>• Industry-aligned curriculum</li>
                <li>• Flexible scheduling options</li>
                <li>• Career services included</li>
                <li>• Financial aid available</li>
              </ul>
            </CardContent>
          </Card>
        </div>
      </div>
    </div>
  );
}
```

#client/src/pages/admin-login.tsx
```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from "@/components/ui/card";
import { useAuth } from "@/hooks/use-auth";
import { useToast } from "@/hooks/use-toast";
import { loginSchema } from "@shared/schema";

type LoginForm = z.infer<typeof loginSchema>;

export default function AdminLogin() {
  const { login, loading } = useAuth();
  const { toast } = useToast();

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (data: LoginForm) => {
    try {
      await login(data.username, data.password);
      toast({
        title: "Login Successful",
        description: "Welcome to the admin dashboard",
      });
    } catch (error) {
      toast({
        title: "Login Failed",
        description: "Invalid username or password",
        variant: "destructive",
      });
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
      <Card className="w-full max-w-md">
        <CardHeader className="text-center">
          <CardTitle className="text-2xl font-bold">Admin Login</CardTitle>
          <CardDescription>
            Access the EduLead Pro admin dashboard
          </CardDescription>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
            <div>
              <Label htmlFor="username">Username</Label>
              <Input
                id="username"
                {...register("username")}
                placeholder="Enter username"
                className={errors.username ? "border-red-500" : ""}
              />
              {errors.username && (
                <p className="text-red-500 text-sm mt-1">{errors.username.message}</p>
              )}
            </div>

            <div>
              <Label htmlFor="password">Password</Label>
              <Input
                id="password"
                type="password"
                {...register("password")}
                placeholder="Enter password"
                className={errors.password ? "border-red-500" : ""}
              />
              {errors.password && (
                <p className="text-red-500 text-sm mt-1">{errors.password.message}</p>
              )}
            </div>

            <Button 
              type="submit" 
              className="w-full" 
              disabled={loading}
            >
              {loading ? "Signing in..." : "Sign In"}
            </Button>
          </form>

          <div className="mt-6 p-4 bg-blue-50 rounded-lg">
            <p className="text-sm text-blue-800 font-medium">Default Credentials:</p>
            <p className="text-sm text-blue-700">Username: admin</p>
            <p className="text-sm text-blue-700">Password: admin123</p>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

#client/src/pages/dashboard.tsx
```tsx
import { useQuery } from "@tanstack/react-query";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Progress } from "@/components/ui/progress";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { useAuth } from "@/hooks/use-auth";
import { Link } from "wouter";
import { 
  Users, 
  TrendingUp, 
  Target, 
  Brain,
  BarChart3,
  UserCheck,
  Clock,
  CheckCircle
} from "lucide-react";

interface Lead {
  id: number;
  name: string;
  email: string;
  phone?: string;
  age?: number;
  location?: string;
  interest: string;
  degreeLevel?: string;
  timeline?: string;
  comments?: string;
  score: number;
  quality: string;
  status: string;
  createdAt: string;
}

interface Stats {
  totalLeads: number;
  highQuality: number;
  mediumQuality: number;
  lowQuality: number;
  conversionRate: number;
  newLeads: number;
  contactedLeads: number;
  qualifiedLeads: number;
  convertedLeads: number;
}

export default function Dashboard() {
  const { user, logout } = useAuth();

  const { data: stats, isLoading: statsLoading } = useQuery({
    queryKey: ["/api/analytics/stats"],
  });

  const { data: recentLeads, isLoading: leadsLoading } = useQuery({
    queryKey: ["/api/leads"],
  });

  const getQualityColor = (quality: string) => {
    switch (quality) {
      case "high": return "bg-green-100 text-green-800";
      case "medium": return "bg-yellow-100 text-yellow-800";
      case "low": return "bg-red-100 text-red-800";
      default: return "bg-gray-100 text-gray-800";
    }
  };

  const getStatusColor = (status: string) => {
    switch (status) {
      case "new": return "bg-blue-100 text-blue-800";
      case "contacted": return "bg-purple-100 text-purple-800";
      case "qualified": return "bg-orange-100 text-orange-800";
      case "converted": return "bg-green-100 text-green-800";
      default: return "bg-gray-100 text-gray-800";
    }
  };

  if (statsLoading || leadsLoading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white shadow-sm border-b">
        <div className="max-w-7xl mx-auto px-4 py-4">
          <div className="flex justify-between items-center">
            <div>
              <h1 className="text-2xl font-bold text-gray-900">Admin Dashboard</h1>
              <p className="text-gray-600">Welcome back, {user?.username}</p>
            </div>
            <div className="flex gap-4">
              <Link href="/leads">
                <Button variant="outline">View All Leads</Button>
              </Link>
              <Button onClick={logout} variant="outline">
                Logout
              </Button>
            </div>
          </div>
        </div>
      </div>

      <div className="max-w-7xl mx-auto px-4 py-8">
        {/* Stats Overview */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
          <Card>
            <CardContent className="p-6">
              <div className="flex items-center">
                <Users className="h-8 w-8 text-blue-600" />
                <div className="ml-4">
                  <p className="text-sm font-medium text-gray-600">Total Leads</p>
                  <p className="text-2xl font-bold text-gray-900">{stats?.totalLeads || 0}</p>
                </div>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardContent className="p-6">
              <div className="flex items-center">
                <TrendingUp className="h-8 w-8 text-green-600" />
                <div className="ml-4">
                  <p className="text-sm font-medium text-gray-600">Conversion Rate</p>
                  <p className="text-2xl font-bold text-gray-900">{stats?.conversionRate || 0}%</p>
                </div>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardContent className="p-6">
              <div className="flex items-center">
                <Target className="h-8 w-8 text-orange-600" />
                <div className="ml-4">
                  <p className="text-sm font-medium text-gray-600">High Quality</p>
                  <p className="text-2xl font-bold text-gray-900">{stats?.highQuality || 0}</p>
                </div>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardContent className="p-6">
              <div className="flex items-center">
                <Brain className="h-8 w-8 text-purple-600" />
                <div className="ml-4">
                  <p className="text-sm font-medium text-gray-600">AI Processed</p>
                  <p className="text-2xl font-bold text-gray-900">{stats?.totalLeads || 0}</p>
                </div>
              </div>
            </CardContent>
          </Card>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
          {/* Lead Quality Distribution */}
          <Card>
            <CardHeader>
              <CardTitle className="flex items-center">
                <BarChart3 className="h-5 w-5 mr-2" />
                Lead Quality Distribution
              </CardTitle>
            </CardHeader>
            <CardContent>
              <div className="space-y-4">
                <div>
                  <div className="flex justify-between mb-2">
                    <span className="text-sm font-medium">High Quality</span>
                    <span className="text-sm text-gray-600">{stats?.highQuality || 0}</span>
                  </div>
                  <Progress 
                    value={stats?.totalLeads ? (stats.highQuality / stats.totalLeads) * 100 : 0} 
                    className="h-2"
                  />
                </div>
                
                <div>
                  <div className="flex justify-between mb-2">
                    <span className="text-sm font-medium">Medium Quality</span>
                    <span className="text-sm text-gray-600">{stats?.mediumQuality || 0}</span>
                  </div>
                  <Progress 
                    value={stats?.totalLeads ? (stats.mediumQuality / stats.totalLeads) * 100 : 0} 
                    className="h-2"
                  />
                </div>
                
                <div>
                  <div className="flex justify-between mb-2">
                    <span className="text-sm font-medium">Low Quality</span>
                    <span className="text-sm text-gray-600">{stats?.lowQuality || 0}</span>
                  </div>
                  <Progress 
                    value={stats?.totalLeads ? (stats.lowQuality / stats.totalLeads) * 100 : 0} 
                    className="h-2"
                  />
                </div>
              </div>
            </CardContent>
          </Card>

          {/* Lead Status Pipeline */}
          <Card>
            <CardHeader>
              <CardTitle className="flex items-center">
                <UserCheck className="h-5 w-5 mr-2" />
                Lead Status Pipeline
              </CardTitle>
            </CardHeader>
            <CardContent>
              <div className="space-y-4">
                <div className="flex items-center justify-between p-3 bg-blue-50 rounded-lg">
                  <div className="flex items-center">
                    <Clock className="h-4 w-4 text-blue-600 mr-2" />
                    <span className="font-medium">New Leads</span>
                  </div>
                  <Badge variant="secondary">{stats?.newLeads || 0}</Badge>
                </div>
                
                <div className="flex items-center justify-between p-3 bg-purple-50 rounded-lg">
                  <div className="flex items-center">
                    <Users className="h-4 w-4 text-purple-600 mr-2" />
                    <span className="font-medium">Contacted</span>
                  </div>
                  <Badge variant="secondary">{stats?.contactedLeads || 0}</Badge>
                </div>
                
                <div className="flex items-center justify-between p-3 bg-orange-50 rounded-lg">
                  <div className="flex items-center">
                    <Target className="h-4 w-4 text-orange-600 mr-2" />
                    <span className="font-medium">Qualified</span>
                  </div>
                  <Badge variant="secondary">{stats?.qualifiedLeads || 0}</Badge>
                </div>
                
                <div className="flex items-center justify-between p-3 bg-green-50 rounded-lg">
                  <div className="flex items-center">
                    <CheckCircle className="h-4 w-4 text-green-600 mr-2" />
                    <span className="font-medium">Converted</span>
                  </div>
                  <Badge variant="secondary">{stats?.convertedLeads || 0}</Badge>
                </div>
              </div>
            </CardContent>
          </Card>
        </div>

        {/* Recent Leads */}
        <Card className="mt-8">
          <CardHeader>
            <CardTitle>Recent Leads</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="overflow-x-auto">
              <table className="w-full">
                <thead>
                  <tr className="border-b">
                    <th className="text-left p-2">Name</th>
                    <th className="text-left p-2">Interest</th>
                    <th className="text-left p-2">AI Score</th>
                    <th className="text-left p-2">Quality</th>
                    <th className="text-left p-2">Status</th>
                    <th className="text-left p-2">Date</th>
                  </tr>
                </thead>
                <tbody>
                  {recentLeads?.slice(0, 10).map((lead: Lead) => (
                    <tr key={lead.id} className="border-b hover:bg-gray-50">
                      <td className="p-2 font-medium">{lead.name}</td>
                      <td className="p-2">{lead.interest}</td>
                      <td className="p-2">
                        <div className="flex items-center">
                          <div className="w-12 bg-gray-200 rounded-full h-2 mr-2">
                            <div 
                              className="bg-blue-600 h-2 rounded-full" 
                              style={{ width: `${lead.score}%` }}
                            ></div>
                          </div>
                          <span className="text-sm">{lead.score}%</span>
                        </div>
                      </td>
                      <td className="p-2">
                        <Badge className={getQualityColor(lead.quality)}>
                          {lead.quality}
                        </Badge>
                      </td>
                      <td className="p-2">
                        <Badge className={getStatusColor(lead.status)}>
                          {lead.status}
                        </Badge>
                      </td>
                      <td className="p-2 text-sm text-gray-600">
                        {new Date(lead.createdAt).toLocaleDateString()}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
            
            <div className="mt-4 text-center">
              <Link href="/leads">
                <Button variant="outline">View All Leads</Button>
              </Link>
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

#client/src/pages/leads.tsx
```tsx
import { useState } from "react";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { useAuth } from "@/hooks/use-auth";
import { useToast } from "@/hooks/use-toast";
import { apiRequest } from "@/lib/queryClient";
import { Link } from "wouter";
import { Search, Filter, ArrowLeft, ExternalLink } from "lucide-react";

interface Lead {
  id: number;
  name: string;
  email: string;
  phone?: string;
  age?: number;
  location?: string;
  interest: string;
  degreeLevel?: string;
  timeline?: string;
  comments?: string;
  score: number;
  quality: string;
  status: string;
  createdAt: string;
}

export default function Leads() {
  const [searchQuery, setSearchQuery] = useState("");
  const [qualityFilter, setQualityFilter] = useState("");
  const [statusFilter, setStatusFilter] = useState("");
  const { logout } = useAuth();
  const { toast } = useToast();
  const queryClient = useQueryClient();

  const { data: leads, isLoading } = useQuery({
    queryKey: ["/api/leads", { search: searchQuery, quality: qualityFilter, status: statusFilter }],
    queryFn: () => {
      const params = new URLSearchParams();
      if (searchQuery) params.append("search", searchQuery);
      if (qualityFilter) params.append("quality", qualityFilter);
      if (statusFilter) params.append("status", statusFilter);
      
      return apiRequest(`/api/leads?${params.toString()}`);
    },
  });

  const updateStatusMutation = useMutation({
    mutationFn: async ({ leadId, status }: { leadId: number; status: string }) => {
      return apiRequest(`/api/leads/${leadId}/status`, {
        method: "PATCH",
        body: JSON.stringify({ status }),
      });
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["/api/leads"] });
      toast({
        title: "Status Updated",
        description: "Lead status has been updated successfully",
      });
    },
    onError: () => {
      toast({
        title: "Update Failed",
        description: "Failed to update lead status",
        variant: "destructive",
      });
    },
  });

  const syncToCRMMutation = useMutation({
    mutationFn: async (leadId: number) => {
      return apiRequest(`/api/crm/sync/${leadId}`, {
        method: "POST",
      });
    },
    onSuccess: () => {
      toast({
        title: "CRM Sync Successful",
        description: "Lead has been synced to CRM successfully",
      });
    },
    onError: () => {
      toast({
        title: "CRM Sync Failed",
        description: "Failed to sync lead to CRM",
        variant: "destructive",
      });
    },
  });

  const getQualityColor = (quality: string) => {
    switch (quality) {
      case "high": return "bg-green-100 text-green-800 border-green-200";
      case "medium": return "bg-yellow-100 text-yellow-800 border-yellow-200";
      case "low": return "bg-red-100 text-red-800 border-red-200";
      default: return "bg-gray-100 text-gray-800 border-gray-200";
    }
  };

  const getStatusColor = (status: string) => {
    switch (status) {
      case "new": return "bg-blue-100 text-blue-800 border-blue-200";
      case "contacted": return "bg-purple-100 text-purple-800 border-purple-200";
      case "qualified": return "bg-orange-100 text-orange-800 border-orange-200";
      case "converted": return "bg-green-100 text-green-800 border-green-200";
      default: return "bg-gray-100 text-gray-800 border-gray-200";
    }
  };

  const handleStatusChange = (leadId: number, newStatus: string) => {
    updateStatusMutation.mutate({ leadId, status: newStatus });
  };

  const handleCRMSync = (leadId: number) => {
    syncToCRMMutation.mutate(leadId);
  };

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white shadow-sm border-b">
        <div className="max-w-7xl mx-auto px-4 py-4">
          <div className="flex justify-between items-center">
            <div className="flex items-center">
              <Link href="/dashboard">
                <Button variant="outline" size="sm" className="mr-4">
                  <ArrowLeft className="h-4 w-4 mr-2" />
                  Dashboard
                </Button>
              </Link>
              <div>
                <h1 className="text-2xl font-bold text-gray-900">Lead Management</h1>
                <p className="text-gray-600">Manage and track all educational leads</p>
              </div>
            </div>
            <Button onClick={logout} variant="outline">
              Logout
            </Button>
          </div>
        </div>
      </div>

      <div className="max-w-7xl mx-auto px-4 py-8">
        {/* Filters */}
        <Card className="mb-6">
          <CardContent className="p-6">
            <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
              <div className="relative">
                <Search className="absolute left-3 top-3 h-4 w-4 text-gray-400" />
                <Input
                  placeholder="Search leads..."
                  value={searchQuery}
                  onChange={(e) => setSearchQuery(e.target.value)}
                  className="pl-9"
                />
              </div>
              
              <Select value={qualityFilter} onValueChange={setQualityFilter}>
                <SelectTrigger>
                  <SelectValue placeholder="Filter by quality" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="">All Qualities</SelectItem>
                  <SelectItem value="high">High Quality</SelectItem>
                  <SelectItem value="medium">Medium Quality</SelectItem>
                  <SelectItem value="low">Low Quality</SelectItem>
                </SelectContent>
              </Select>
              
              <Select value={statusFilter} onValueChange={setStatusFilter}>
                <SelectTrigger>
                  <SelectValue placeholder="Filter by status" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="">All Statuses</SelectItem>
                  <SelectItem value="new">New</SelectItem>
                  <SelectItem value="contacted">Contacted</SelectItem>
                  <SelectItem value="qualified">Qualified</SelectItem>
                  <SelectItem value="converted">Converted</SelectItem>
                </SelectContent>
              </Select>
              
              <Button
                variant="outline"
                onClick={() => {
                  setSearchQuery("");
                  setQualityFilter("");
                  setStatusFilter("");
                }}
              >
                <Filter className="h-4 w-4 mr-2" />
                Clear Filters
              </Button>
            </div>
          </CardContent>
        </Card>

        {/* Leads Table */}
        <Card>
          <CardHeader>
            <CardTitle>
              All Leads ({leads?.length || 0})
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="overflow-x-auto">
              <table className="w-full">
                <thead>
                  <tr className="border-b bg-gray-50">
                    <th className="text-left p-3 font-semibold">Lead Info</th>
                    <th className="text-left p-3 font-semibold">Interest & Level</th>
                    <th className="text-left p-3 font-semibold">AI Score</th>
                    <th className="text-left p-3 font-semibold">Quality</th>
                    <th className="text-left p-3 font-semibold">Status</th>
                    <th className="text-left p-3 font-semibold">Timeline</th>
                    <th className="text-left p-3 font-semibold">Actions</th>
                  </tr>
                </thead>
                <tbody>
                  {leads?.map((lead: Lead) => (
                    <tr key={lead.id} className="border-b hover:bg-gray-50">
                      <td className="p-3">
                        <div>
                          <div className="font-medium text-gray-900">{lead.name}</div>
                          <div className="text-sm text-gray-600">{lead.email}</div>
                          {lead.phone && (
                            <div className="text-sm text-gray-600">{lead.phone}</div>
                          )}
                          {lead.location && (
                            <div className="text-sm text-gray-500">{lead.location}</div>
                          )}
                        </div>
                      </td>
                      
                      <td className="p-3">
                        <div>
                          <div className="font-medium">{lead.interest}</div>
                          {lead.degreeLevel && (
                            <div className="text-sm text-gray-600 capitalize">
                              {lead.degreeLevel.replace('-', ' ')}
                            </div>
                          )}
                        </div>
                      </td>
                      
                      <td className="p-3">
                        <div className="flex items-center">
                          <div className="w-16 bg-gray-200 rounded-full h-2 mr-3">
                            <div 
                              className={`h-2 rounded-full ${
                                lead.score >= 80 ? 'bg-green-500' :
                                lead.score >= 60 ? 'bg-yellow-500' : 'bg-red-500'
                              }`}
                              style={{ width: `${lead.score}%` }}
                            ></div>
                          </div>
                          <span className="text-sm font-medium">{lead.score}%</span>
                        </div>
                      </td>
                      
                      <td className="p-3">
                        <Badge className={getQualityColor(lead.quality)}>
                          {lead.quality}
                        </Badge>
                      </td>
                      
                      <td className="p-3">
                        <Select
                          value={lead.status}
                          onValueChange={(value) => handleStatusChange(lead.id, value)}
                        >
                          <SelectTrigger className="w-32">
                            <SelectValue />
                          </SelectTrigger>
                          <SelectContent>
                            <SelectItem value="new">New</SelectItem>
                            <SelectItem value="contacted">Contacted</SelectItem>
                            <SelectItem value="qualified">Qualified</SelectItem>
                            <SelectItem value="converted">Converted</SelectItem>
                          </SelectContent>
                        </Select>
                      </td>
                      
                      <td className="p-3">
                        <div className="text-sm">
                          {lead.timeline && (
                            <div className="capitalize">{lead.timeline.replace('-', ' ')}</div>
                          )}
                          <div className="text-gray-500">
                            {new Date(lead.createdAt).toLocaleDateString()}
                          </div>
                        </div>
                      </td>
                      
                      <td className="p-3">
                        <div className="flex gap-2">
                          {lead.quality === "high" && (
                            <Button
                              size="sm"
                              variant="outline"
                              onClick={() => handleCRMSync(lead.id)}
                              disabled={syncToCRMMutation.isPending}
                            >
                              <ExternalLink className="h-3 w-3 mr-1" />
                              CRM
                            </Button>
                          )}
                        </div>
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
            
            {leads?.length === 0 && (
              <div className="text-center py-8">
                <p className="text-gray-500">No leads found matching your criteria.</p>
              </div>
            )}
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

#client/src/pages/not-found.tsx
```tsx
import { Link } from "wouter";
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Home, ArrowLeft } from "lucide-react";

export default function NotFound() {
  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
      <Card className="w-full max-w-md text-center">
        <CardHeader>
          <div className="mx-auto mb-4 text-6xl">🎓</div>
          <CardTitle className="text-2xl font-bold">Page Not Found</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-gray-600 mb-6">
            Sorry, we couldn't find the page you're looking for.
          </p>
          
          <div className="space-y-3">
            <Link href="/">
              <Button className="w-full">
                <Home className="h-4 w-4 mr-2" />
                Go to Lead Form
              </Button>
            </Link>
            
            <Link href="/admin-login">
              <Button variant="outline" className="w-full">
                <ArrowLeft className="h-4 w-4 mr-2" />
                Admin Login
              </Button>
            </Link>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```
