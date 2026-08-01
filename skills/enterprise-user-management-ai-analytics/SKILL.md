---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "build an enterprise user management dashboard"
  - "integrate AI analytics for user behavior tracking"
  - "create a task management system with burnout detection"
  - "implement JWT authentication with role-based access"
  - "set up AI-powered ticket classification system"
  - "develop predictive analytics for project management"
  - "build a kanban board with time tracking"
  - "create an admin dashboard with anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: JWT-based authentication, role-based access control (Admin/User)
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Ticket creation, assignment, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Tools**: User CRUD operations, audit logs, organization-wide analytics

The system uses a React frontend, Node.js/Express backend, MongoDB database, and FastAPI ML service for AI features.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB running locally or connection string

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
```

Start ML service:

```bash
uvicorn main:app --reload
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

**POST** `/api/auth/register`
```javascript
// Register new user
fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'john.doe',
    email: 'john@company.com',
    password: 'SecurePass123',
    role: 'user' // or 'admin'
  })
});
```

**POST** `/api/auth/login`
```javascript
// Login and get JWT token
const response = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@company.com',
    password: 'SecurePass123'
  })
});
const { token, user } = await response.json();
// Store token for subsequent requests
localStorage.setItem('authToken', token);
```

### User Management (Backend)

**GET** `/api/users` (Admin only)
```javascript
// Get all users
const token = localStorage.getItem('authToken');
const response = await fetch('http://localhost:5000/api/users', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const users = await response.json();
```

**PUT** `/api/users/:userId`
```javascript
// Update user
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    username: 'john.doe.updated',
    role: 'admin'
  })
});
```

**DELETE** `/api/users/:userId`
```javascript
// Delete user (Admin only)
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Bearer ${token}` }
});
```

### Task Management

**POST** `/api/tasks`
```javascript
// Create task
const task = await fetch('http://localhost:5000/api/tasks', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement user authentication',
    description: 'Add JWT-based auth to API',
    assignedTo: userId,
    priority: 'high',
    status: 'todo', // todo, in-progress, done
    dueDate: '2026-05-01'
  })
}).then(r => r.json());
```

**PATCH** `/api/tasks/:taskId/status`
```javascript
// Update task status (Kanban board)
await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ status: 'in-progress' })
});
```

**POST** `/api/tasks/:taskId/time`
```javascript
// Track time on task
await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    duration: 3600, // seconds
    date: new Date().toISOString()
  })
});
```

### Support Tickets

**POST** `/api/tickets`
```javascript
// Create support ticket
const ticket = await fetch('http://localhost:5000/api/tickets', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Cannot access reports',
    description: 'Getting 403 error when accessing analytics page',
    priority: 'medium',
    category: 'technical'
  })
}).then(r => r.json());
```

**GET** `/api/tickets`
```javascript
// Get all tickets (filtered by role)
const tickets = await fetch('http://localhost:5000/api/tickets', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(r => r.json());
```

### AI/ML Endpoints

**POST** `/classify-ticket`
```javascript
// AI-based ticket classification
const classification = await fetch('http://localhost:8000/classify-ticket', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Server keeps crashing',
    description: 'Production server crashes every 2 hours with memory error'
  })
}).then(r => r.json());
// Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'devops-team' }
```

**POST** `/predict-risk`
```javascript
// Predict user risk based on behavior
const risk = await fetch('http://localhost:8000/predict-risk', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    loginAttempts: 5,
    failedLogins: 3,
    accessPatterns: ['unusual_time', 'new_location'],
    activityScore: 0.3
  })
}).then(r => r.json());
// Returns: { riskLevel: 'high', confidence: 0.87, reasons: [...] }
```

**POST** `/detect-anomaly`
```javascript
// Detect anomalous behavior
const anomaly = await fetch('http://localhost:8000/detect-anomaly', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    action: 'bulk_data_download',
    timestamp: new Date().toISOString(),
    metadata: { fileCount: 500, size: 5000000000 }
  })
}).then(r => r.json());
// Returns: { isAnomaly: true, score: 0.92, alert: true }
```

**POST** `/analyze-burnout`
```javascript
// Analyze employee burnout risk
const burnout = await fetch('http://localhost:8000/analyze-burnout', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    weeklyHours: 65,
    tasksCompleted: 45,
    tasksOverdue: 8,
    workloadTrend: 'increasing'
  })
}).then(r => r.json());
// Returns: { burnoutRisk: 'high', recommendation: 'redistribute_tasks' }
```

**POST** `/predict-project-delay`
```javascript
// Predict project delays
const prediction = await fetch('http://localhost:8000/predict-project-delay', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: projectId,
    tasksTotal: 50,
    tasksCompleted: 15,
    daysRemaining: 10,
    teamSize: 5,
    complexityScore: 0.8
  })
}).then(r => r.json());
// Returns: { delayProbability: 0.78, estimatedDelay: 7, suggestions: [...] }
```

## Frontend Components

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('authToken'));

  useEffect(() => {
    if (token) {
      // Verify token and fetch user
      fetch(`${process.env.REACT_APP_API_URL}/auth/me`, {
        headers: { 'Authorization': `Bearer ${token}` }
      })
        .then(r => r.json())
        .then(data => setUser(data.user))
        .catch(() => logout());
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('authToken', data.token);
    setToken(data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('authToken');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const allTasks = await response.json();
    
    setTasks({
      todo: allTasks.filter(t => t.status === 'todo'),
      'in-progress': allTasks.filter(t => t.status === 'in-progress'),
      done: allTasks.filter(t => t.status === 'done')
    });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AIAnalytics = () => {
  const { token, user } = useAuth();
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    // Fetch burnout analysis
    const burnout = await fetch(`${process.env.REACT_APP_ML_URL}/analyze-burnout`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: user._id,
        weeklyHours: user.weeklyHours || 40,
        tasksCompleted: user.tasksCompleted || 0,
        tasksOverdue: user.tasksOverdue || 0
      })
    }).then(r => r.json());

    // Fetch risk prediction
    const risk = await fetch(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: user._id,
        loginAttempts: user.loginAttempts || 0,
        failedLogins: user.failedLogins || 0
      })
    }).then(r => r.json());

    setAnalytics({ burnout, risk });
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="burnout-analysis">
        <h3>Burnout Risk: {analytics.burnout.burnoutRisk}</h3>
        <p>Recommendation: {analytics.burnout.recommendation}</p>
      </div>

      <div className="risk-analysis">
        <h3>Security Risk: {analytics.risk.riskLevel}</h3>
        <p>Confidence: {(analytics.risk.confidence * 100).toFixed(1)}%</p>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No authentication token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.userId
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tasks (user sees only their tasks, admin sees all)
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.userId };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### Ticket Classification

```python
# ml-service/classifiers/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

class TicketClassifier:
    def __init__(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.vectorizer = pickle.load(open(f'{model_path}/vectorizer.pkl', 'rb'))
            self.model = pickle.load(open(f'{model_path}/ticket_model.pkl', 'rb'))
        except:
            self.vectorizer = TfidfVectorizer(max_features=1000)
            self.model = MultinomialNB()
            self.is_trained = False
    
    def classify(self, title, description):
        text = f"{title} {description}"
        features = self.vectorizer.transform([text])
        category = self.model.predict(features)[0]
        priority = self._predict_priority(text)
        
        return {
            'category': category,
            'priority': priority,
            'confidence': max(self.model.predict_proba(features)[0])
        }
    
    def _predict_priority(self, text):
        urgent_keywords = ['urgent', 'critical', 'down', 'crash', 'emergency']
        high_keywords = ['error', 'bug', 'issue', 'problem']
        
        text_lower = text.lower()
        if any(kw in text_lower for kw in urgent_keywords):
            return 'urgent'
        elif any(kw in text_lower for kw in high_keywords):
            return 'high'
        return 'medium'
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from classifiers.ticket_classifier import TicketClassifier
from analyzers.burnout_analyzer import BurnoutAnalyzer
from analyzers.risk_predictor import RiskPredictor
import os

app = FastAPI()

# Initialize ML models
ticket_classifier = TicketClassifier()
burnout_analyzer = BurnoutAnalyzer()
risk_predictor = RiskPredictor()

class TicketInput(BaseModel):
    title: str
    description: str

class BurnoutInput(BaseModel):
    userId: str
    weeklyHours: float
    tasksCompleted: int
    tasksOverdue: int
    workloadTrend: str = "stable"

class RiskInput(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    accessPatterns: list = []
    activityScore: float = 1.0

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    try:
        result = ticket_classifier.classify(ticket.title, ticket.description)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(data: BurnoutInput):
    try:
        result = burnout_analyzer.analyze(
            data.weeklyHours,
            data.tasksCompleted,
            data.tasksOverdue,
            data.workloadTrend
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskInput):
    try:
        result = risk_predictor.predict(
            data.loginAttempts,
            data.failedLogins,
            data.accessPatterns,
            data.activityScore
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "ml_service": "running"}
```

## Configuration

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginAttempts: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  lastLogin: Date,
  weeklyHours: { type: Number, default: 0 },
  tasksCompleted: { type: Number, default: 0 },
  tasksOverdue: { type: Number, default: 0 }
}, { timestamps: true });

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useAuth();

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Real-time Notifications

```javascript
// frontend/src/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import { useAuth } from './useAuth';

export const useNotifications = () => {
  const { token } = useAuth();
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/notifications`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, [token]);

  return notifications;
};
```

## Troubleshooting

### JWT Token Expired

```javascript
// frontend/src/utils/api.js
const apiRequest = async (url, options = {}) => {
  const token = localStorage.getItem('authToken');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });

  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('authToken');
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  return response.json();
};
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```python
# ml-service/main.py
# Add timeout and retry logic
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Health check for debugging
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "models_loaded": {
            "ticket_classifier": hasattr(ticket_classifier, 'model'),
            "burnout_analyzer": hasattr(burnout_analyzer, 'model')
        }
    }
```

### Task Status Not Updating

Check that the task ID is valid and user has permissions:

```javascript
// backend/routes/tasks.js
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    // Check permissions
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user.userId) {
      return res.status(403).json({ error: 'Not authorized to update this task' });
    }

    task.status = req.body.status;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```
