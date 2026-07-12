---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management dashboard with AI"
  - "implement task tracking with burnout detection"
  - "create admin panel with anomaly detection"
  - "add AI-powered ticket classification"
  - "deploy user management system with ML"
  - "configure JWT authentication for user system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-driven insights. It provides:

- **User Management**: Role-based access control, authentication (JWT), and user CRUD operations
- **Task Management**: Kanban boards, time tracking, and assignment workflows
- **Support Tickets**: Smart ticket classification and routing using AI
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Real-time analytics, audit logs, and organizational insights

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js/Express) - REST APIs and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML models and predictions

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

Create `.env` file in `backend/`:

```env
PORT=5000
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
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

Create `.env` file in `ml-service/`:

```env
MONGO_URI=your_mongodb_connection_string
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Reference

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securePassword123',
    role: 'user' // or 'admin'
  })
});
const data = await response.json();
// Returns: { token, user: { id, name, email, role } }
```

**Login**
```javascript
// POST /api/auth/login
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'securePassword123'
  })
});
const { token, user } = await response.json();
localStorage.setItem('token', token);
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
});
const users = await response.json();
```

**Update User**
```javascript
// PUT /api/users/:userId
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Updated Name',
    role: 'admin',
    status: 'active'
  })
});
```

**Delete User**
```javascript
// DELETE /api/users/:userId
await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'DELETE',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
});
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement new feature',
    description: 'Add user analytics dashboard',
    assignedTo: 'userId123',
    priority: 'high', // low, medium, high
    status: 'todo', // todo, in-progress, done
    dueDate: '2026-05-01T00:00:00Z',
    estimatedHours: 8
  })
});
const task = await response.json();
```

**Get User Tasks**
```javascript
// GET /api/tasks?userId=userId123&status=in-progress
const params = new URLSearchParams({
  userId: 'userId123',
  status: 'in-progress'
});
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks?${params}`, {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
});
const tasks = await response.json();
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:taskId/status
await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'done',
    timeSpent: 6.5 // hours
  })
});
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    subject: 'Login Issue',
    description: 'Unable to login with correct credentials',
    priority: 'high',
    category: 'technical' // technical, billing, general
  })
});
const ticket = await response.json();
```

**Get Tickets**
```javascript
// GET /api/tickets?status=open&priority=high
const response = await fetch(
  `${process.env.REACT_APP_API_URL}/api/tickets?status=open&priority=high`,
  {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('token')}`
    }
  }
);
const tickets = await response.json();
```

## ML Service API Reference

### AI Ticket Classification

**Classify Ticket**
```javascript
// POST /api/ml/classify-ticket
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    subject: 'Cannot access my account',
    description: 'I forgot my password and the reset email is not arriving',
    userId: 'user123'
  })
});
const classification = await response.json();
// Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'supportTeam' }
```

### Risk Prediction

**Predict User Risk**
```javascript
// POST /api/ml/predict-risk
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    loginAttempts: 5,
    lastLoginHour: 3, // 3 AM
    loginLocation: 'unknown',
    activityScore: 0.2
  })
});
const risk = await response.json();
// Returns: { riskLevel: 'high', score: 0.85, factors: ['unusual_time', 'unknown_location'] }
```

### Anomaly Detection

**Detect Anomalies**
```javascript
// POST /api/ml/detect-anomaly
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-anomaly`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    activityData: {
      tasksCompleted: 15,
      hoursWorked: 60,
      loginCount: 25,
      dataAccessed: 500 // MB
    },
    timeframe: 'week'
  })
});
const anomaly = await response.json();
// Returns: { isAnomaly: true, anomalyScore: 0.92, reason: 'excessive_hours' }
```

### Burnout Detection

**Analyze Burnout Risk**
```javascript
// POST /api/ml/burnout-analysis
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    workloadData: {
      hoursWorked: 55,
      tasksAssigned: 12,
      tasksCompleted: 8,
      overdueCount: 3,
      weekendWork: true,
      avgTaskCompletionTime: 8.5 // hours
    }
  })
});
const burnout = await response.json();
// Returns: { burnoutRisk: 'high', score: 0.78, recommendations: ['reduce_workload', 'redistribute_tasks'] }
```

### Predictive Project Insights

**Predict Project Delay**
```javascript
// POST /api/ml/predict-delay
const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-delay`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: 'proj123',
    totalTasks: 50,
    completedTasks: 20,
    daysElapsed: 30,
    totalDays: 60,
    teamSize: 5,
    averageVelocity: 0.7 // tasks per day
  })
});
const prediction = await response.json();
// Returns: { willDelay: true, estimatedDelayDays: 10, completionProbability: 0.65 }
```

## Common Integration Patterns

### React Component with Task Management

```javascript
import React, { useState, useEffect } from 'react';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
  };

  return (
    <div className="kanban-board">
      <TaskColumn title="To Do" tasks={tasks.todo} onMove={updateTaskStatus} />
      <TaskColumn title="In Progress" tasks={tasks.inProgress} onMove={updateTaskStatus} />
      <TaskColumn title="Done" tasks={tasks.done} onMove={updateTaskStatus} />
    </div>
  );
};

export default TaskBoard;
```

### Admin Dashboard with AI Analytics

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAnalytics();
    checkRisks();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${API_URL}/api/admin/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const checkRisks = async () => {
    const token = localStorage.getItem('token');
    
    // Get all users
    const usersResponse = await fetch(`${API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersResponse.json();

    // Check each user for burnout
    const burnoutChecks = await Promise.all(
      users.map(async (user) => {
        const response = await fetch(`${ML_URL}/api/ml/burnout-analysis`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: user.id,
            workloadData: user.workloadStats
          })
        });
        const result = await response.json();
        return { user: user.name, ...result };
      })
    );

    const highRisk = burnoutChecks.filter(b => b.burnoutRisk === 'high');
    setAlerts(highRisk);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="stats-grid">
          <StatCard title="Total Users" value={analytics.totalUsers} />
          <StatCard title="Active Tasks" value={analytics.activeTasks} />
          <StatCard title="Open Tickets" value={analytics.openTickets} />
        </div>
      )}

      {alerts.length > 0 && (
        <div className="alerts">
          <h2>Burnout Risk Alerts</h2>
          {alerts.map((alert, idx) => (
            <div key={idx} className="alert-item">
              {alert.user} - Risk Score: {alert.score.toFixed(2)}
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

### Automated Ticket Classification

```javascript
// Backend middleware for automatic ticket classification
const classifyTicket = async (req, res, next) => {
  const { subject, description } = req.body;
  const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

  try {
    const response = await fetch(`${ML_SERVICE_URL}/api/ml/classify-ticket`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        subject,
        description,
        userId: req.user.id
      })
    });

    const classification = await response.json();
    
    // Add AI classification to request
    req.ticketClassification = {
      category: classification.category,
      priority: classification.priority,
      suggestedAssignee: classification.suggestedAssignee
    };

    next();
  } catch (error) {
    // Fallback to manual classification
    req.ticketClassification = {
      category: 'general',
      priority: 'medium',
      suggestedAssignee: null
    };
    next();
  }
};

// Use in route
app.post('/api/tickets', authenticate, classifyTicket, async (req, res) => {
  const ticket = await Ticket.create({
    ...req.body,
    ...req.ticketClassification,
    createdBy: req.user.id
  });
  res.json(ticket);
});
```

## Configuration

### MongoDB Models

**User Model Example**
```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  lastLogin: Date,
  workloadStats: {
    hoursWorked: { type: Number, default: 0 },
    tasksCompleted: { type: Number, default: 0 },
    tasksAssigned: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

**Task Model Example**
```javascript
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  estimatedHours: Number,
  timeSpent: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### JWT Middleware

```javascript
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };
```

## Troubleshooting

### CORS Issues

If frontend can't connect to backend:

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Errors

Verify ML service is running and accessible:

```bash
curl http://localhost:8000/health
```

If connection fails, check:
- ML service is running on correct port
- Environment variable `ML_SERVICE_URL` is set correctly
- Firewall isn't blocking port 8000

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Token Expiration

Handle expired tokens gracefully:

```javascript
// frontend/utils/api.js
const apiCall = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(`${process.env.REACT_APP_API_URL}${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      ...options.headers
    }
  });

  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    return;
  }

  return response.json();
};
```

### ML Model Not Found

If ML predictions fail:

```python
# ml-service/main.py
import os
from pathlib import Path

MODEL_PATH = Path(os.getenv('MODEL_PATH', './models'))
MODEL_PATH.mkdir(exist_ok=True)

# Initialize or load models
def load_or_train_model(model_name):
    model_file = MODEL_PATH / f'{model_name}.pkl'
    if model_file.exists():
        return joblib.load(model_file)
    else:
        # Train new model with default data
        model = train_default_model(model_name)
        joblib.dump(model, model_file)
        return model
```

This skill enables AI agents to assist developers in deploying, configuring, and extending the Enterprise User Management System with AI Analytics across all three layers: frontend UI, backend APIs, and ML services.
