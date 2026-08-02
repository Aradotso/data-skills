---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI ticket classification system"
  - "build kanban board with time tracking"
  - "integrate burnout detection analytics"
  - "configure JWT authentication for user management"
  - "deploy enterprise management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines role-based access control, task management, support ticketing, and AI-powered analytics. The system provides intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics to improve organizational productivity.

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - User and admin dashboards with Kanban boards and analytics
2. **Backend** (Node.js) - REST API for user management, authentication, and data operations
3. **ML Service** (FastAPI + scikit-learn) - AI models for predictions and analytics

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication

**User Registration**

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    })
  });
  return response.json();
};
```

**User Login**

```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

**Authenticated Requests**

```javascript
const getAuthHeaders = () => ({
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${localStorage.getItem('token')}`
});
```

### User Management (Admin Only)

**Get All Users**

```javascript
// GET /api/users
const getAllUsers = async () => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: getAuthHeaders()
  });
  return response.json();
};
```

**Create User**

```javascript
// POST /api/users
const createUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      role: userData.role,
      department: userData.department
    })
  });
  return response.json();
};
```

**Update User**

```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: getAuthHeaders(),
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User**

```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: getAuthHeaders()
  });
  return response.json();
};
```

### Task Management

**Get User Tasks**

```javascript
// GET /api/tasks
const getUserTasks = async () => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: getAuthHeaders()
  });
  return response.json();
};
```

**Create Task**

```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo, // user ID
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate,
      estimatedHours: taskData.estimatedHours
    })
  });
  return response.json();
};
```

**Update Task Status**

```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: getAuthHeaders(),
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

**Log Time Entry**

```javascript
// POST /api/tasks/:id/time
const logTimeEntry = async (taskId, hours, description) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      hours: hours,
      description: description,
      date: new Date().toISOString()
    })
  });
  return response.json();
};
```

### Support Tickets

**Create Ticket**

```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'hr', 'general'
    })
  });
  return response.json();
};
```

**Get Tickets**

```javascript
// GET /api/tickets
const getTickets = async (filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: getAuthHeaders()
  });
  return response.json();
};
```

**Update Ticket**

```javascript
// PATCH /api/tickets/:id
const updateTicket = async (ticketId, updates) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: getAuthHeaders(),
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// POST /classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: 'Ticket title'
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return data;
};
```

### Risk Prediction

```javascript
// POST /predict-risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 45,
        failed_logins: 3,
        tasks_overdue: 2,
        avg_completion_time: 8.5
      }
    })
  });
  const data = await response.json();
  // Returns: { risk_level: 'medium', score: 0.65, factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.loginTime,
      ip_address: userActivity.ipAddress,
      location: userActivity.location,
      device: userActivity.device
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, reason: 'Unusual login location' }
  return data;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout
const detectBurnout = async (userId, timeframe = 30) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      days: timeframe,
      metrics: {
        hours_worked: 180,
        tasks_completed: 25,
        avg_task_duration: 7.2,
        overtime_hours: 40
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// POST /predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      avg_velocity: projectData.avgVelocity,
      team_size: projectData.teamSize,
      deadline: projectData.deadline
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.72, estimated_delay_days: 5, suggestions: [...] }
  return data;
};
```

## React Component Patterns

### Protected Route Component

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requireAdmin && userRole !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
      headers: getAuthHeaders()
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: getAuthHeaders(),
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  const stopTimer = () => setIsRunning(false);
  
  const saveTimeEntry = async () => {
    const hours = (seconds / 3600).toFixed(2);
    await logTimeEntry(taskId, parseFloat(hours), 'Work session');
    setSeconds(0);
    setIsRunning(false);
  };

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <h2>{formatTime(seconds)}</h2>
      <button onClick={startTimer} disabled={isRunning}>Start</button>
      <button onClick={stopTimer} disabled={!isRunning}>Stop</button>
      <button onClick={saveTimeEntry} disabled={isRunning || seconds === 0}>
        Save
      </button>
    </div>
  );
};

export default TimeTracker;
```

### AI Insights Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AIInsightsDashboard = () => {
  const [insights, setInsights] = useState({
    burnoutRisk: null,
    riskPrediction: null,
    anomalies: []
  });

  useEffect(() => {
    loadInsights();
  }, []);

  const loadInsights = async () => {
    const userId = localStorage.getItem('userId');
    
    // Fetch burnout detection
    const burnout = await detectBurnout(userId, 30);
    
    // Fetch risk prediction
    const risk = await predictUserRisk(userId);
    
    setInsights({
      burnoutRisk: burnout,
      riskPrediction: risk,
      anomalies: []
    });
  };

  return (
    <div className="ai-insights">
      <h2>AI-Powered Insights</h2>
      
      {insights.burnoutRisk && (
        <div className="insight-card">
          <h3>Burnout Risk</h3>
          <div className={`risk-level ${insights.burnoutRisk.burnout_risk}`}>
            {insights.burnoutRisk.burnout_risk.toUpperCase()}
          </div>
          <p>Score: {(insights.burnoutRisk.score * 100).toFixed(0)}%</p>
          <ul>
            {insights.burnoutRisk.recommendations?.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
      
      {insights.riskPrediction && (
        <div className="insight-card">
          <h3>Security Risk</h3>
          <div className={`risk-level ${insights.riskPrediction.risk_level}`}>
            {insights.riskPrediction.risk_level.toUpperCase()}
          </div>
          <p>Score: {(insights.riskPrediction.score * 100).toFixed(0)}%</p>
        </div>
      )}
    </div>
  );
};

export default AIInsightsDashboard;
```

## Database Schema Patterns

### User Schema (MongoDB)

```javascript
const userSchema = {
  name: String,
  email: { type: String, unique: true },
  password: String, // hashed
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  lastLogin: Date,
  isActive: { type: Boolean, default: true },
  profile: {
    avatar: String,
    phone: String,
    position: String
  }
};
```

### Task Schema

```javascript
const taskSchema = {
  title: String,
  description: String,
  assignedTo: { type: ObjectId, ref: 'User' },
  createdBy: { type: ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'inprogress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  estimatedHours: Number,
  actualHours: Number,
  timeEntries: [{
    hours: Number,
    description: String,
    date: Date,
    loggedBy: { type: ObjectId, ref: 'User' }
  }],
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
};
```

### Ticket Schema

```javascript
const ticketSchema = {
  title: String,
  description: String,
  createdBy: { type: ObjectId, ref: 'User' },
  assignedTo: { type: ObjectId, ref: 'User' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'], default: 'medium' },
  category: { type: String, enum: ['technical', 'hr', 'general', 'other'] },
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  comments: [{
    text: String,
    createdBy: { type: ObjectId, ref: 'User' },
    createdAt: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
};
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# File Upload
MAX_FILE_SIZE=5242880
UPLOAD_PATH=./uploads
```

### ML Service Configuration

```env
# Model Settings
MODEL_PATH=./models
CLASSIFICATION_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml_service.log

# Performance
BATCH_SIZE=32
MAX_WORKERS=4
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
REACT_APP_WEBSOCKET_URL=ws://localhost:5000
```

## Common Workflows

### Complete User Onboarding Flow

```javascript
const onboardNewUser = async (userData) => {
  try {
    // 1. Create user account
    const user = await createUser({
      name: userData.name,
      email: userData.email,
      role: 'user',
      department: userData.department
    });
    
    // 2. Assign initial tasks
    await createTask({
      title: 'Complete onboarding checklist',
      description: 'Review company policies and complete profile',
      assignedTo: user._id,
      priority: 'high',
      dueDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
    });
    
    // 3. Send welcome notification
    await sendNotification(user._id, {
      type: 'welcome',
      message: `Welcome to the team, ${user.name}!`
    });
    
    return user;
  } catch (error) {
    console.error('Onboarding error:', error);
    throw error;
  }
};
```

### AI-Assisted Ticket Processing

```javascript
const processTicketWithAI = async (ticketData) => {
  try {
    // 1. Classify ticket using AI
    const classification = await classifyTicket(ticketData.description);
    
    // 2. Create ticket with AI suggestions
    const ticket = await createTicket({
      title: ticketData.title,
      description: ticketData.description,
      priority: classification.priority,
      category: classification.category,
      aiClassification: {
        suggestedCategory: classification.category,
        suggestedPriority: classification.priority,
        confidence: classification.confidence
      }
    });
    
    // 3. Auto-assign based on category
    const assignee = await findBestAssignee(classification.category);
    if (assignee) {
      await updateTicket(ticket._id, { assignedTo: assignee._id });
    }
    
    return ticket;
  } catch (error) {
    console.error('AI ticket processing error:', error);
    throw error;
  }
};
```

### Performance Analytics Workflow

```javascript
const generateUserPerformanceReport = async (userId, days = 30) => {
  try {
    // 1. Fetch user tasks
    const tasks = await getUserTasks();
    
    // 2. Calculate metrics
    const completedTasks = tasks.filter(t => t.status === 'done');
    const totalHours = tasks.reduce((sum, t) => sum + (t.actualHours || 0), 0);
    const avgCompletionTime = completedTasks.reduce((sum, t) => 
      sum + (t.completedAt - t.createdAt), 0) / completedTasks.length;
    
    // 3. Get AI insights
    const burnout = await detectBurnout(userId, days);
    const risk = await predictUserRisk(userId);
    
    // 4. Compile report
    return {
      userId,
      period: `Last ${days} days`,
      metrics: {
        tasksCompleted: completedTasks.length,
        totalHours,
        avgCompletionTime: avgCompletionTime / (1000 * 60 * 60), // hours
        productivity: (completedTasks.length / tasks.length * 100).toFixed(1)
      },
      aiInsights: {
        burnoutRisk: burnout.burnout_risk,
        securityRisk: risk.risk_level
      },
      recommendations: [...burnout.recommendations, ...risk.factors]
    };
  } catch (error) {
    console.error('Report generation error:', error);
    throw error;
  }
};
```

## Troubleshooting

### JWT Authentication Issues

```javascript
// Clear invalid tokens
const clearAuth = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('userRole');
  localStorage.removeItem('userId');
  window.location.href = '/login';
};

// Token expiration handler
const handleTokenExpiration = (error) => {
  if (error.message.includes('jwt expired') || error.status === 401) {
    clearAuth();
  }
};

// Use in fetch wrapper
const authenticatedFetch = async (url, options = {}) => {
  try {
    const response = await fetch(url, {
      ...options,
      headers: { ...getAuthHeaders(), ...options.headers }
    });
    
    if (response.status === 401) {
      clearAuth();
      throw new Error('Session expired');
    }
    
    return response;
  } catch (error) {
    handleTokenExpiration(error);
    throw error;
  }
};
```

### ML Service Connection Issues

```javascript
// Health check for ML service
const checkMLServiceHealth = async () => {
  try {
    const response = await fetch('http://localhost:8000/health', {
      timeout: 5000
    });
    return response.ok;
  } catch (error) {
    console.error('ML Service unavailable:', error);
    return false;
  }
};

// Fallback for AI features
const safeClassifyTicket = async (ticketText) => {
  const isMLAvailable = await checkMLServiceHealth();
  
  if (!isMLAvailable) {
    // Return default classification
    return {
      category: 'general',
      priority: 'medium',
      confidence: 0,
      fallback: true
    };
  }
  
  return classifyTicket(ticketText);
};
```

### Database Connection Issues

```javascript
// MongoDB connection retry logic
const connectWithRetry = async (uri, retries = 5) => {
  for (let i = 0; i < retries; i++) {
    try {
      await mongoose.connect(uri, {
        useNewUrlParser: true,
        useUnifiedTopology: true,
        serverSelectionTimeoutMS: 5000
      });
      console.log('MongoDB connected');
      return;
    } catch (error) {
      console.error(`Connection attempt ${i + 1} failed:`, error.message);
      if (i < retries - 1) {
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }
  throw new Error('Could not connect to MongoDB');
};
```

### CORS Issues

```javascript
// Backend CORS configuration
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

## Testing

### API Testing Example

```javascript
// Test user authentication
const testAuth = async () => {
  try {
    // Register
    const registerRes = await registerUser({
      name: 'Test User',
      email: 'test@example.com',
      password: 'TestPass123!'
    });
    console.log('Registration:', registerRes);
    
    // Login
    const loginRes = await loginUser('test@example.com', 'TestPass123!');
    console.log('Login:', loginRes);
    
    // Fetch profile
    const profileRes = await fetch('http://localhost:5000/api/users/me', {
      headers: getAuthHeaders()
    });
    console.log('Profile:', await profileRes.json());
    
  } catch (error) {
    console.error('Auth test failed:', error);
  }
};
```

### ML Model Testing

```javascript
// Test AI classification
const testMLClassification = async () => {
  const testCases = [
    'Server is down and users cannot access the application',
    'Need vacation approval for next week',
    'How do I reset my password?'
  ];
  
  for (const text of testCases) {
    const result = await classifyTicket(text);
    console.log(`Text: "${text}"`);
    console.log(`Category: ${result.category}, Priority: ${result.priority}, Confidence: ${result.confidence}`);
    console.log('---');
  }
};
```

## Deployment Considerations

### Production Environment Variables

```bash
# Backend
export NODE_ENV=production
export MONGODB_URI="mongodb+srv://user:pass@cluster.mongodb.net/db"
export JWT_SECRET="$(openssl rand -base64 32)"

# Frontend build
npm run build
# Serve from build directory
```

### Docker Deployment

```dockerfile
# Backend Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json
