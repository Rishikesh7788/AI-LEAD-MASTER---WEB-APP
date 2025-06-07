#client/src/components/ai-scoring.tsx
```tsx
import { Badge } from "@/components/ui/badge";
import { Progress } from "@/components/ui/progress";
import { Card, CardContent } from "@/components/ui/card";
import { Brain, TrendingUp, Target } from "lucide-react";
import { cn } from "@/lib/utils";

interface AIScoringProps {
  score: number;
  quality: "high" | "medium" | "low";
  prediction?: {
    confidence: number;
    factors: string[];
  };
  className?: string;
}

export function AIScoring({ score, quality, prediction, className }: AIScoringProps) {
  const getQualityConfig = (quality: string) => {
    switch (quality) {
      case "high":
        return {
          color: "bg-green-100 text-green-800 border-green-200",
          bgColor: "bg-green-50",
          progressColor: "bg-green-500",
          icon: <Target className="h-5 w-5 text-green-600" />,
        };
      case "medium":
        return {
          color: "bg-yellow-100 text-yellow-800 border-yellow-200",
          bgColor: "bg-yellow-50",
          progressColor: "bg-yellow-500",
          icon: <TrendingUp className="h-5 w-5 text-yellow-600" />,
        };
      case "low":
        return {
          color: "bg-red-100 text-red-800 border-red-200",
          bgColor: "bg-red-50",
          progressColor: "bg-red-500",
          icon: <Brain className="h-5 w-5 text-red-600" />,
        };
      default:
        return {
          color: "bg-gray-100 text-gray-800 border-gray-200",
          bgColor: "bg-gray-50",
          progressColor: "bg-gray-500",
          icon: <Brain className="h-5 w-5 text-gray-600" />,
        };
    }
  };

  const config = getQualityConfig(quality);

  const formatFactor = (factor: string) => {
    return factor
      .split('_')
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ');
  };

  return (
    <Card className={cn("w-full", className)}>
      <CardContent className={cn("p-6", config.bgColor)}>
        <div className="space-y-4">
          {/* Header */}
          <div className="flex items-center justify-between">
            <div className="flex items-center space-x-2">
              <Brain className="h-6 w-6 text-blue-600" />
              <h3 className="text-lg font-semibold">AI Lead Analysis</h3>
            </div>
            <Badge className={config.color}>
              {quality.toUpperCase()} QUALITY
            </Badge>
          </div>

          {/* Score Display */}
          <div className="space-y-2">
            <div className="flex items-center justify-between">
              <span className="text-sm font-medium">Lead Score</span>
              <span className="text-2xl font-bold">{score}%</span>
            </div>
            <Progress value={score} className="h-3" />
          </div>

          {/* Confidence */}
          {prediction && (
            <div className="space-y-2">
              <div className="flex items-center justify-between">
                <span className="text-sm font-medium">AI Confidence</span>
                <span className="text-sm font-semibold">
                  {Math.round(prediction.confidence * 100)}%
                </span>
              </div>
              <Progress value={prediction.confidence * 100} className="h-2" />
            </div>
          )}

          {/* Analysis Factors */}
          {prediction && prediction.factors.length > 0 && (
            <div>
              <h4 className="text-sm font-medium mb-2">Key Factors</h4>
              <div className="flex flex-wrap gap-2">
                {prediction.factors.slice(0, 4).map((factor, index) => (
                  <Badge
                    key={index}
                    variant="outline"
                    className="text-xs"
                  >
                    {formatFactor(factor)}
                  </Badge>
                ))}
              </div>
            </div>
          )}

          {/* Score Interpretation */}
          <div className="text-xs text-gray-600 bg-white/50 p-3 rounded-lg">
            {score >= 85 && (
              <p>Excellent prospect with high conversion probability. Recommend immediate contact.</p>
            )}
            {score >= 70 && score < 85 && (
              <p>Strong candidate with good potential. Follow up within 24 hours.</p>
            )}
            {score >= 50 && score < 70 && (
              <p>Moderate interest level. Consider nurturing campaign approach.</p>
            )}
            {score < 50 && (
              <p>Lower engagement likelihood. May benefit from educational content first.</p>
            )}
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#client/src/components/chatbot.tsx
```tsx
import { useState, useRef, useEffect } from "react";
import { useMutation } from "@tanstack/react-query";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { ScrollArea } from "@/components/ui/scroll-area";
import { apiRequest } from "@/lib/queryClient";
import { MessageCircle, Send, Bot, User } from "lucide-react";

interface Message {
  role: "user" | "bot";
  content: string;
  timestamp: Date;
}

export function Chatbot() {
  const [messages, setMessages] = useState<Message[]>([
    {
      role: "bot",
      content: "Hi! I'm here to help you learn about our educational programs. What field are you interested in exploring?",
      timestamp: new Date(),
    },
  ]);
  const [inputValue, setInputValue] = useState("");
  const [sessionId] = useState(() => `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`);
  const scrollAreaRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  const sendMessageMutation = useMutation({
    mutationFn: async (message: string) => {
      const response = await apiRequest("/api/chat/message", {
        method: "POST",
        body: JSON.stringify({
          sessionId,
          message,
          leadData: null,
        }),
      });
      return response;
    },
    onSuccess: (data) => {
      setMessages(prev => [
        ...prev,
        {
          role: "bot",
          content: data.response,
          timestamp: new Date(),
        },
      ]);
    },
    onError: () => {
      setMessages(prev => [
        ...prev,
        {
          role: "bot",
          content: "I'm sorry, I'm having trouble responding right now. Please try again in a moment.",
          timestamp: new Date(),
        },
      ]);
    },
  });

  const handleSendMessage = (e: React.FormEvent) => {
    e.preventDefault();
    if (!inputValue.trim()) return;

    const userMessage: Message = {
      role: "user",
      content: inputValue.trim(),
      timestamp: new Date(),
    };

    setMessages(prev => [...prev, userMessage]);
    sendMessageMutation.mutate(inputValue.trim());
    setInputValue("");
  };

  // Auto-scroll to bottom when new messages arrive
  useEffect(() => {
    if (scrollAreaRef.current) {
      const scrollContainer = scrollAreaRef.current.querySelector('[data-radix-scroll-area-viewport]');
      if (scrollContainer) {
        scrollContainer.scrollTop = scrollContainer.scrollHeight;
      }
    }
  }, [messages]);

  // Focus input when component mounts
  useEffect(() => {
    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, []);

  const formatTime = (date: Date) => {
    return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  };

  return (
    <Card className="w-full h-96 flex flex-col">
      <CardHeader className="pb-3">
        <CardTitle className="flex items-center text-lg">
          <MessageCircle className="h-5 w-5 mr-2 text-blue-600" />
          Chat Assistant
        </CardTitle>
      </CardHeader>
      
      <CardContent className="flex-1 flex flex-col p-4 pt-0">
        {/* Messages Area */}
        <ScrollArea ref={scrollAreaRef} className="flex-1 pr-4">
          <div className="space-y-4">
            {messages.map((message, index) => (
              <div
                key={index}
                className={`flex ${message.role === "user" ? "justify-end" : "justify-start"}`}
              >
                <div className={`flex max-w-[80%] ${message.role === "user" ? "flex-row-reverse" : "flex-row"}`}>
                  {/* Avatar */}
                  <div className={`flex-shrink-0 ${message.role === "user" ? "ml-2" : "mr-2"}`}>
                    <div className={`w-8 h-8 rounded-full flex items-center justify-center ${
                      message.role === "user" 
                        ? "bg-blue-500 text-white" 
                        : "bg-gray-100 text-gray-600"
                    }`}>
                      {message.role === "user" ? (
                        <User className="h-4 w-4" />
                      ) : (
                        <Bot className="h-4 w-4" />
                      )}
                    </div>
                  </div>
                  
                  {/* Message Content */}
                  <div className={`rounded-lg px-3 py-2 ${
                    message.role === "user"
                      ? "bg-blue-500 text-white"
                      : "bg-gray-100 text-gray-900"
                  }`}>
                    <p className="text-sm">{message.content}</p>
                    <p className={`text-xs mt-1 ${
                      message.role === "user" ? "text-blue-100" : "text-gray-500"
                    }`}>
                      {formatTime(message.timestamp)}
                    </p>
                  </div>
                </div>
              </div>
            ))}
            
            {/* Typing Indicator */}
            {sendMessageMutation.isPending && (
              <div className="flex justify-start">
                <div className="flex mr-2">
                  <div className="w-8 h-8 rounded-full bg-gray-100 flex items-center justify-center">
                    <Bot className="h-4 w-4 text-gray-600" />
                  </div>
                </div>
                <div className="bg-gray-100 rounded-lg px-3 py-2">
                  <div className="flex space-x-1">
                    <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></div>
                    <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.1s' }}></div>
                    <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.2s' }}></div>
                  </div>
                </div>
              </div>
            )}
          </div>
        </ScrollArea>

        {/* Input Area */}
        <div className="mt-4">
          <form onSubmit={handleSendMessage} className="flex space-x-2">
            <Input
              ref={inputRef}
              value={inputValue}
              onChange={(e) => setInputValue(e.target.value)}
              placeholder="Ask about our programs..."
              disabled={sendMessageMutation.isPending}
              className="flex-1"
            />
            <Button
              type="submit"
              size="sm"
              disabled={!inputValue.trim() || sendMessageMutation.isPending}
            >
              <Send className="h-4 w-4" />
            </Button>
          </form>
        </div>
      </CardContent>
    </Card>
  );
}
```

#client/src/hooks/use-auth.tsx
```tsx
import { createContext, useContext, useState, useEffect, ReactNode } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { apiRequest } from "@/lib/queryClient";
import { authStorage, isTokenExpired } from "@/lib/auth";

interface User {
  id: number;
  username: string;
  role: string;
}

interface AuthContextType {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (username: string, password: string) => Promise<void>;
  logout: () => void;
  loading: boolean;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const queryClient = useQueryClient();

  // Initialize auth state from localStorage
  useEffect(() => {
    const initializeAuth = async () => {
      const savedToken = authStorage.getToken();
      const savedUser = authStorage.getUser();

      if (savedToken && savedUser && !isTokenExpired(savedToken)) {
        try {
          // Verify token with server
          await apiRequest("/api/auth/verify", {
            method: "POST",
            headers: {
              Authorization: `Bearer ${savedToken}`,
            },
          });
          
          setToken(savedToken);
          setUser(savedUser);
        } catch (error) {
          // Token is invalid, clear storage
          authStorage.clear();
        }
      } else {
        // Token expired or invalid, clear storage
        authStorage.clear();
      }
      
      setLoading(false);
    };

    initializeAuth();
  }, []);

  const login = async (username: string, password: string) => {
    setLoading(true);
    try {
      const response = await apiRequest("/api/auth/login", {
        method: "POST",
        body: JSON.stringify({ username, password }),
      });

      const { token: newToken, user: newUser } = response;
      
      setToken(newToken);
      setUser(newUser);
      
      authStorage.setToken(newToken);
      authStorage.setUser(newUser);
    } catch (error) {
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    authStorage.clear();
    queryClient.clear();
  };

  const isAuthenticated = Boolean(token && user);

  return (
    <AuthContext.Provider
      value={{
        user,
        token,
        isAuthenticated,
        login,
        logout,
        loading,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

#client/src/hooks/use-toast.ts
```tsx
import * as React from "react"
import type { ToastActionElement, ToastProps } from "@/components/ui/toast"

const TOAST_LIMIT = 1
const TOAST_REMOVE_DELAY = 1000000

type ToasterToast = ToastProps & {
  id: string
  title?: React.ReactNode
  description?: React.ReactNode
  action?: ToastActionElement
}

const actionTypes = {
  ADD_TOAST: "ADD_TOAST",
  UPDATE_TOAST: "UPDATE_TOAST",
  DISMISS_TOAST: "DISMISS_TOAST",
  REMOVE_TOAST: "REMOVE_TOAST",
} as const

let count = 0

function genId() {
  count = (count + 1) % Number.MAX_SAFE_INTEGER
  return count.toString()
}

type ActionType = typeof actionTypes

type Action =
  | {
      type: ActionType["ADD_TOAST"]
      toast: ToasterToast
    }
  | {
      type: ActionType["UPDATE_TOAST"]
      toast: Partial<ToasterToast>
    }
  | {
      type: ActionType["DISMISS_TOAST"]
      toastId?: ToasterToast["id"]
    }
  | {
      type: ActionType["REMOVE_TOAST"]
      toastId?: ToasterToast["id"]
    }

interface State {
  toasts: ToasterToast[]
}

const toastTimeouts = new Map<string, ReturnType<typeof setTimeout>>()

const addToRemoveQueue = (toastId: string) => {
  if (toastTimeouts.has(toastId)) {
    return
  }

  const timeout = setTimeout(() => {
    toastTimeouts.delete(toastId)
    dispatch({
      type: "REMOVE_TOAST",
      toastId: toastId,
    })
  }, TOAST_REMOVE_DELAY)

  toastTimeouts.set(toastId, timeout)
}

export const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case "ADD_TOAST":
      return {
        ...state,
        toasts: [action.toast, ...state.toasts].slice(0, TOAST_LIMIT),
      }

    case "UPDATE_TOAST":
      return {
        ...state,
        toasts: state.toasts.map((t) =>
          t.id === action.toast.id ? { ...t, ...action.toast } : t
        ),
      }

    case "DISMISS_TOAST": {
      const { toastId } = action

      if (toastId) {
        addToRemoveQueue(toastId)
      } else {
        state.toasts.forEach((toast) => {
          addToRemoveQueue(toast.id)
        })
      }

      return {
        ...state,
        toasts: state.toasts.map((t) =>
          t.id === toastId || toastId === undefined
            ? {
                ...t,
                open: false,
              }
            : t
        ),
      }
    }
    case "REMOVE_TOAST":
      if (action.toastId === undefined) {
        return {
          ...state,
          toasts: [],
        }
      }
      return {
        ...state,
        toasts: state.toasts.filter((t) => t.id !== action.toastId),
      }
  }
}

const listeners: Array<(state: State) => void> = []

let memoryState: State = { toasts: [] }

function dispatch(action: Action) {
  memoryState = reducer(memoryState, action)
  listeners.forEach((listener) => {
    listener(memoryState)
  })
}

type Toast = Omit<ToasterToast, "id">

function toast({ ...props }: Toast) {
  const id = genId()

  const update = (props: ToasterToast) =>
    dispatch({
      type: "UPDATE_TOAST",
      toast: { ...props, id },
    })
  const dismiss = () => dispatch({ type: "DISMISS_TOAST", toastId: id })

  dispatch({
    type: "ADD_TOAST",
    toast: {
      ...props,
      id,
      open: true,
      onOpenChange: (open) => {
        if (!open) dismiss()
      },
    },
  })

  return {
    id: id,
    dismiss,
    update,
  }
}

function useToast() {
  const [state, setState] = React.useState<State>(memoryState)

  React.useEffect(() => {
    listeners.push(setState)
    return () => {
      const index = listeners.indexOf(setState)
      if (index > -1) {
        listeners.splice(index, 1)
      }
    }
  }, [state])

  return {
    ...state,
    toast,
    dismiss: (toastId?: string) => dispatch({ type: "DISMISS_TOAST", toastId }),
  }
}

export { useToast, toast }
```

#client/src/lib/utils.ts
```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

#client/src/lib/auth.ts
```typescript
interface User {
  id: number;
  username: string;
  role: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
}

export const authStorage = {
  getToken(): string | null {
    try {
      return localStorage.getItem("auth_token");
    } catch {
      return null;
    }
  },

  setToken(token: string): void {
    try {
      localStorage.setItem("auth_token", token);
    } catch {
      // Handle localStorage errors silently
    }
  },

  removeToken(): void {
    try {
      localStorage.removeItem("auth_token");
    } catch {
      // Handle localStorage errors silently
    }
  },

  getUser(): User | null {
    try {
      const userData = localStorage.getItem("auth_user");
      return userData ? JSON.parse(userData) : null;
    } catch {
      return null;
    }
  },

  setUser(user: User): void {
    try {
      localStorage.setItem("auth_user", JSON.stringify(user));
    } catch {
      // Handle localStorage errors silently
    }
  },

  removeUser(): void {
    try {
      localStorage.removeItem("auth_user");
    } catch {
      // Handle localStorage errors silently
    }
  },

  clear(): void {
    this.removeToken();
    this.removeUser();
  },
};

export const getAuthHeaders = (): Record<string, string> => {
  const token = authStorage.getToken();
  return token ? { Authorization: `Bearer ${token}` } : {};
};

export const isTokenExpired = (token: string): boolean => {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const currentTime = Date.now() / 1000;
    return payload.exp < currentTime;
  } catch {
    return true;
  }
};

export const initializeAuth = (): AuthState => {
  const token = authStorage.getToken();
  const user = authStorage.getUser();
  
  if (token && user && !isTokenExpired(token)) {
    return {
      user,
      token,
      isAuthenticated: true,
    };
  }

  authStorage.clear();
  return {
    user: null,
    token: null,
    isAuthenticated: false,
  };
};
```

#client/src/lib/queryClient.ts
```typescript
import { QueryClient } from "@tanstack/react-query";
import { getAuthHeaders } from "./auth";

async function throwIfResNotOk(res: Response) {
  if (!res.ok) {
    const errorText = await res.text();
    let errorMessage = res.statusText;
    
    try {
      const errorJson = JSON.parse(errorText);
      errorMessage = errorJson.message || errorText;
    } catch {
      errorMessage = errorText || res.statusText;
    }
    
    throw new Error(errorMessage);
  }
}

export async function apiRequest(
  url: string,
  options: RequestInit = {}
): Promise<any> {
  const { headers = {}, ...restOptions } = options;
  
  const authHeaders = getAuthHeaders();
  
  const response = await fetch(url, {
    ...restOptions,
    headers: {
      "Content-Type": "application/json",
      ...authHeaders,
      ...headers,
    },
  });

  await throwIfResNotOk(response);
  
  const contentType = response.headers.get("content-type");
  if (contentType && contentType.includes("application/json")) {
    return response.json();
  }
  
  return response.text();
}

type UnauthorizedBehavior = "returnNull" | "throw";

export const getQueryFn: <T>(options: {
  on401: UnauthorizedBehavior;
}) => (context: { queryKey: readonly unknown[] }) => Promise<T | null> = ({
  on401,
}) => {
  return async ({ queryKey }) => {
    const url = queryKey[0] as string;
    
    try {
      return await apiRequest(url);
    } catch (error: any) {
      if (error.message.includes("401") || error.message.includes("Unauthorized")) {
        if (on401 === "returnNull") {
          return null;
        }
      }
      throw error;
    }
  };
};

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      queryFn: getQueryFn({ on401: "throw" }),
      retry: (failureCount, error: any) => {
        if (error?.message?.includes("401")) {
          return false;
        }
        return failureCount < 2;
      },
    },
  },
});
```
