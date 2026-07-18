---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task tracking, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "create user dashboard with AI analytics"
  - "implement AI-powered ticket classification"
  - "add burnout detection to user management"
  - "build task tracking with kanban board"
  - "configure ML service for user analytics"
  - "integrate AI risk prediction system"
  - "deploy user management with FastAPI ML backend"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user/admin dashboards, task management with Kanban boards, support ticket systems, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - User and admin dashboards with task tracking
2. **Backend** (Node.js/Express) - REST API, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Installation

### Prerequisites

- Node.js 14+
- Python 3.8+
- MongoDB 4.4+

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

Create `.env` file:

```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-users
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```bash
ML_SERVICE_PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key Backend API Endpoints

### Authentication

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users
const fetchAllUsers = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      status: 'todo', // todo, inProgress, done
      priority: taskData.priority,
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## AI/ML Service API

### Risk Prediction

```javascript
// POST /predict/risk - Predict user risk score
const predictUserRisk = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.id,
      taskCompletionRate: userData.taskCompletionRate,
      averageTaskTime: userData.averageTaskTime,
      missedDeadlines: userData.missedDeadlines,
      loginFrequency: userData.loginFrequency,
      lastActivityDays: userData.lastActivityDays
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /detect/anomaly - Detect unusual user behavior
const detectAnomaly = async (behaviorData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect/anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: behaviorData.userId,
      loginTime: behaviorData.loginTime,
      ipAddress: behaviorData.ipAddress,
      actionsPerMinute: behaviorData.actionsPerMinute,
      dataAccessPattern: behaviorData.dataAccessPattern
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, confidence: 0.89, anomalyType: "unusual_access" }
  return result;
};
```

### Burnout Detection

```javascript
// POST /analyze/burnout - Analyze employee burnout risk
const analyzeBurnout = async (workloadData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/analyze/burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      weeklyHours: workloadData.weeklyHours,
      overtimeHours: workloadData.overtimeHours,
      taskLoad: workloadData.taskLoad,
      deadlinePressure: workloadData.deadlinePressure,
      workLifeBalance: workloadData.workLifeBalance
    })
  });
  const result = await response.json();
  // Returns: { burnoutScore: 0.68, level: "moderate", recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /classify/ticket - Auto-classify support tickets
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify/ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // Returns: { category: "technical", priority: "high", suggestedAssignee: "IT-Team" }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /predict/project - Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/project`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      tasksCompleted: projectData.tasksCompleted,
      totalTasks: projectData.totalTasks,
      averageTaskTime: projectData.averageTaskTime,
      remainingDays: projectData.remainingDays,
      teamSize: projectData.teamSize
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.72, expectedDelay: 5, bottlenecks: [...] }
  return result;
};
```

## React Component Patterns

### Protected Route with JWT

```javascript
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

// Usage in App.js
<Route 
  path="/admin" 
  element={
    <ProtectedRoute requiredRole="admin">
      <AdminDashboard />
    </ProtectedRoute>
  } 
/>
```

### Kanban Board Component

```javascript
import { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserTasks();
  }, [userId]);

  const fetchUserTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'inProgress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    fetchUserTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} />
    </div>
  );
};
```

### AI Risk Dashboard Component

```javascript
import { useState, useEffect } from 'react';

const RiskDashboard = ({ users }) => {
  const [riskScores, setRiskScores] = useState({});

  useEffect(() => {
    analyzeAllUsers();
  }, [users]);

  const analyzeAllUsers = async () => {
    const scores = {};
    
    for (const user of users) {
      const response = await fetch(
        `${process.env.REACT_APP_ML_URL}/predict/risk`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: user.id,
            taskCompletionRate: user.stats.completionRate,
            averageTaskTime: user.stats.avgTaskTime,
            missedDeadlines: user.stats.missedDeadlines,
            loginFrequency: user.stats.loginFrequency,
            lastActivityDays: user.stats.lastActivityDays
          })
        }
      );
      const risk = await response.json();
      scores[user.id] = risk;
    }
    
    setRiskScores(scores);
  };

  return (
    <div className="risk-dashboard">
      <h2>User Risk Analysis</h2>
      {users.map(user => (
        <div key={user.id} className={`risk-card ${riskScores[user.id]?.riskLevel}`}>
          <h3>{user.name}</h3>
          <p>Risk Score: {riskScores[user.id]?.riskScore?.toFixed(2)}</p>
          <p>Level: {riskScores[user.id]?.riskLevel}</p>
          <ul>
            {riskScores[user.id]?.factors?.map((factor, i) => (
              <li key={i}>{factor}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
};
```

## Database Schema Examples

### User Model (MongoDB)

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  },
  stats: {
    taskCompletionRate: { type: Number, default: 0 },
    averageTaskTime: { type: Number, default: 0 },
    missedDeadlines: { type: Number, default: 0 },
    loginFrequency: { type: Number, default: 0 },
    lastActivityDays: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'inProgress', 'done'], 
    default: 'todo' 
  },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'] },
  category: { type: String, enum: ['technical', 'billing', 'general', 'hr'] },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedAssignee: String
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Backend Environment Variables

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-users

# JWT
JWT_SECRET=your_secure_jwt_secret_change_in_production
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Environment Variables

```bash
# Service
ML_SERVICE_PORT=8000
LOG_LEVEL=INFO

# Models
MODEL_PATH=./models
MODEL_RETRAIN_INTERVAL=86400

# Database (for ML logs)
ML_DB_URI=mongodb://localhost:27017/ml-analytics

# Feature flags
ENABLE_ONLINE_LEARNING=true
ENABLE_AUTO_RETRAIN=true
```

### Frontend Environment Variables

```bash
# API URLs
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000

# Feature flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_ANALYTICS=true

# Environment
REACT_APP_ENV=development
```

## Common Workflows

### User Registration and Login Flow

```javascript
// Registration
const handleRegister = async (formData) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(formData)
    });
    
    if (!response.ok) throw new Error('Registration failed');
    
    const data = await response.json();
    console.log('User registered:', data.user);
    
    // Auto-login after registration
    return handleLogin({ email: formData.email, password: formData.password });
  } catch (error) {
    console.error('Registration error:', error);
  }
};

// Login
const handleLogin = async (credentials) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    
    if (!response.ok) throw new Error('Login failed');
    
    const data = await response.json();
    
    // Store auth data
    localStorage.setItem('token', data.token);
    localStorage.setItem('user', JSON.stringify(data.user));
    
    // Redirect based on role
    if (data.user.role === 'admin') {
      window.location.href = '/admin';
    } else {
      window.location.href = '/dashboard';
    }
  } catch (error) {
    console.error('Login error:', error);
  }
};
```

### Admin User Management Workflow

```javascript
const AdminUserManagement = () => {
  const [users, setUsers] = useState([]);
  const [selectedUser, setSelectedUser] = useState(null);
  const token = localStorage.getItem('token');

  // Fetch all users
  const loadUsers = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setUsers(data);
  };

  // Add new user
  const addUser = async (userData) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(userData)
    });
    await loadUsers();
  };

  // Update user
  const updateUser = async (userId, updates) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(updates)
    });
    await loadUsers();
  };

  // Delete user
  const deleteUser = async (userId) => {
    if (!window.confirm('Are you sure?')) return;
    
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${token}` }
    });
    await loadUsers();
  };

  // Check user risk score
  const checkUserRisk = async (user) => {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/risk`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: user.id,
        taskCompletionRate: user.stats.taskCompletionRate,
        averageTaskTime: user.stats.averageTaskTime,
        missedDeadlines: user.stats.missedDeadlines,
        loginFrequency: user.stats.loginFrequency,
        lastActivityDays: user.stats.lastActivityDays
      })
    });
    const risk = await response.json();
    alert(`Risk Level: ${risk.riskLevel} (${risk.riskScore.toFixed(2)})`);
  };

  return (
    <div>
      <button onClick={() => addUser({ /* user data */ })}>Add User</button>
      {users.map(user => (
        <div key={user.id}>
          <span>{user.username} ({user.role})</span>
          <button onClick={() => updateUser(user.id, { /* updates */ })}>Edit</button>
          <button onClick={() => deleteUser(user.id)}>Delete</button>
          <button onClick={() => checkUserRisk(user)}>Check Risk</button>
        </div>
      ))}
    </div>
  );
};
```

### Task Tracking Workflow

```javascript
// Complete task tracking cycle
const TaskTracker = ({ userId }) => {
  const [tasks, setTasks] = useState([]);
  const [activeTask, setActiveTask] = useState(null);
  const [timeElapsed, setTimeElapsed] = useState(0);
  const token = localStorage.getItem('token');

  // Start timer for task
  const startTask = (task) => {
    setActiveTask(task);
    const interval = setInterval(() => {
      setTimeElapsed(prev => prev + 1);
    }, 1000);
    
    // Store interval ID for cleanup
    task.intervalId = interval;
  };

  // Stop timer and update task
  const stopTask = async () => {
    if (!activeTask) return;
    
    clearInterval(activeTask.intervalId);
    
    // Update task with time tracked
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${activeTask.id}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        timeTracked: (activeTask.timeTracked || 0) + timeElapsed
      })
    });
    
    setActiveTask(null);
    setTimeElapsed(0);
  };

  // Complete task
  const completeTask = async (taskId) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        status: 'done',
        completedAt: new Date()
      })
    });
    
    // Reload tasks
    loadTasks();
  };

  return (
    <div>
      {activeTask && (
        <div>
          <p>Working on: {activeTask.title}</p>
          <p>Time: {Math.floor(timeElapsed / 60)}m {timeElapsed % 60}s</p>
          <button onClick={stopTask}>Stop</button>
        </div>
      )}
      {tasks.map(task => (
        <div key={task.id}>
          <h3>{task.title}</h3>
          <button onClick={() => startTask(task)}>Start</button>
          <button onClick={() => completeTask(task.id)}>Complete</button>
        </div>
      ))}
    </div>
  );
};
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Add axios interceptor to handle token refresh
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/health

# Check logs
cd ml-service
tail -f logs/ml-service.log

# Restart service with debug mode
uvicorn main:app --reload --log-level debug
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Model Training Performance

```python
# ml-service/utils/optimize.py
from sklearn.model_selection import cross_val_score
import joblib

def optimize_model(X, y, model):
    """Use cross-validation to tune model"""
    scores = cross_val_score(model, X, y, cv=5)
    print(f"Model accuracy: {scores.mean():.2f} (+/- {scores.std():.2f})")
    
    # Save best model
    model.fit(X, y)
    joblib.dump(model, 'models/optimized_model.pkl')
    
    return model
```

### Frontend Build Issues

```bash
# Clear cache and rebuild
cd frontend
rm -rf node_modules package-lock.json
npm install
npm run build

# Check for environment variables
echo $REACT_APP_API_URL
echo $REACT_APP_ML_URL
```

### Database Indexing for Performance

```javascript
// Add indexes to improve query performance
const userSchema = new mongoose.Schema({...});

userSchema.index({ email: 1 });
userSchema.index({ username: 1 });
userSchema.index({ 'stats.taskCompletionRate': -1 });

const taskSchema = new mongoose.Schema({...});

taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ dueDate: 1 });
```

## Testing

### Backend API Testing

```javascript
// tests/auth.test.js
const request = require('supertest');
const app = require('../server');

describe('Auth API', () => {
  it
