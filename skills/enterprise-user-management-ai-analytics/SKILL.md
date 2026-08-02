---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task tracking for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task tracking with AI insights"
  - "build admin dashboard with anomaly detection"
  - "integrate AI ticket classification system"
  - "add burnout detection to user management"
  - "develop user management with predictive analytics"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket handling with AI-powered insights. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights to improve organizational efficiency.

**Architecture:**
- **Frontend**: React.js application with user/admin dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI service with scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB 4.4+

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
MONGO_URI=mongodb://localhost:27017/enterprise-ums
API_KEY=${ML_API_KEY}
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key Components

### Backend API Endpoints

**Authentication:**
```javascript
// POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "SecurePass123!",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "SecurePass123!"
}
```

**User Management (Admin):**
```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user
```

**Task Management:**
```javascript
// POST /api/tasks - Create task
{
  "title": "Implement feature X",
  "description": "Build new analytics dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// GET /api/tasks - Get all tasks
// PUT /api/tasks/:id - Update task status
// GET /api/tasks/user/:userId - Get user tasks
```

**Support Tickets:**
```javascript
// POST /api/tickets - Create ticket
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
```

### Frontend Usage Patterns

**Protected Route Setup:**
```javascript
// src/components/PrivateRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const PrivateRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default PrivateRoute;
```

**API Service Configuration:**
```javascript
// src/services/api.js
import axios from 'axios';

const API = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000',
});

// Add auth token to requests
API.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token expiration
API.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default API;
```

**Task Management Component:**
```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import API from '../services/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await API.get('/api/tasks');
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await API.put(`/api/tasks/${taskId}`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };
  
  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
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

export default TaskBoard;
```

### ML Service Integration

**AI Ticket Classification:**
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str
    
class RiskAnalysisRequest(BaseModel):
    userId: str
    loginAttempts: int
    taskCompletionRate: float
    avgResponseTime: float

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """Classify support ticket using ML"""
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Simple keyword-based classification (replace with trained model)
        categories = {
            'technical': ['login', 'error', 'bug', 'crash', 'issue'],
            'account': ['password', 'access', 'permission', 'account'],
            'feature': ['request', 'enhancement', 'improve', 'add'],
            'general': []
        }
        
        text_lower = text.lower()
        for category, keywords in categories.items():
            if any(keyword in text_lower for keyword in keywords):
                return {
                    "category": category,
                    "confidence": 0.85,
                    "priority": "high" if category == "technical" else "medium"
                }
        
        return {"category": "general", "confidence": 0.5, "priority": "low"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-analysis")
async def analyze_risk(request: RiskAnalysisRequest):
    """Analyze user risk score"""
    risk_score = 0
    factors = []
    
    # Login attempts analysis
    if request.loginAttempts > 5:
        risk_score += 30
        factors.append("High login attempts")
    
    # Task completion analysis
    if request.taskCompletionRate < 0.5:
        risk_score += 25
        factors.append("Low task completion rate")
    
    # Response time analysis
    if request.avgResponseTime > 48:  # hours
        risk_score += 20
        factors.append("Slow response time")
    
    risk_level = "high" if risk_score > 50 else "medium" if risk_score > 25 else "low"
    
    return {
        "riskScore": risk_score,
        "riskLevel": risk_level,
        "factors": factors,
        "recommendation": get_recommendation(risk_level)
    }

def get_recommendation(risk_level: str) -> str:
    recommendations = {
        "high": "Immediate attention required. Review user activity and consider account restrictions.",
        "medium": "Monitor user activity closely. Schedule review if pattern continues.",
        "low": "Normal activity. Continue standard monitoring."
    }
    return recommendations.get(risk_level, "No recommendation")

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: dict):
    """Detect employee burnout based on work patterns"""
    hours_worked = data.get('hoursWorked', 0)
    tasks_completed = data.get('tasksCompleted', 0)
    missed_deadlines = data.get('missedDeadlines', 0)
    
    burnout_score = 0
    
    if hours_worked > 50:
        burnout_score += 40
    if tasks_completed < 5:
        burnout_score += 30
    if missed_deadlines > 2:
        burnout_score += 30
    
    return {
        "burnoutScore": burnout_score,
        "status": "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low",
        "suggestions": [
            "Reduce workload" if hours_worked > 50 else "Maintain balance",
            "Take breaks regularly",
            "Discuss with manager if needed"
        ]
    }
```

**Frontend ML Integration:**
```javascript
// src/services/mlService.js
import axios from 'axios';

const ML_API = axios.create({
  baseURL: process.env.REACT_APP_ML_API_URL || 'http://localhost:8000',
});

export const classifyTicket = async (title, description) => {
  try {
    const response = await ML_API.post('/api/ml/classify-ticket', {
      title,
      description
    });
    return response.data;
  } catch (error) {
    console.error('ML classification failed:', error);
    return { category: 'general', confidence: 0, priority: 'low' };
  }
};

export const analyzeUserRisk = async (userId, userData) => {
  try {
    const response = await ML_API.post('/api/ml/risk-analysis', {
      userId,
      ...userData
    });
    return response.data;
  } catch (error) {
    console.error('Risk analysis failed:', error);
    return null;
  }
};

export const detectBurnout = async (workData) => {
  try {
    const response = await ML_API.post('/api/ml/burnout-detection', workData);
    return response.data;
  } catch (error) {
    console.error('Burnout detection failed:', error);
    return null;
  }
};
```

### Backend Implementation Patterns

**User Model (MongoDB):**
```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  },
  analytics: {
    loginAttempts: { type: Number, default: 0 },
    lastLogin: Date,
    tasksCompleted: { type: Number, default: 0 },
    tasksAssigned: { type: Number, default: 0 }
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Authentication Middleware:**
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);
    
    if (!user || !user.isActive) {
      return res.status(401).json({ error: 'User not found or inactive' });
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};

const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

**Task Controller:**
```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user._id,
      priority: priority || 'medium',
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    
    // Update user analytics
    await User.findByIdAndUpdate(assignedTo, {
      $inc: { 'analytics.tasksAssigned': 1 }
    });
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const task = await Task.findById(id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    // Track completion
    if (updates.status === 'done' && task.status !== 'done') {
      await User.findByIdAndUpdate(task.assignedTo, {
        $inc: { 'analytics.tasksCompleted': 1 }
      });
    }
    
    Object.assign(task, updates);
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    
    const tasks = await Task.find({ assignedTo: userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## Configuration

### Environment Variables

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
MONGO_URI=mongodb://localhost:27017/enterprise-ums
API_KEY=your-ml-api-key
LOG_LEVEL=INFO
```

## Common Workflows

### Creating a New User (Admin)
```javascript
// Frontend component
const createUser = async (userData) => {
  try {
    const response = await API.post('/api/users', {
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user',
      profile: {
        firstName: userData.firstName,
        lastName: userData.lastName,
        department: userData.department,
        position: userData.position
      }
    });
    
    console.log('User created:', response.data);
    return response.data;
  } catch (error) {
    console.error('Failed to create user:', error.response?.data);
    throw error;
  }
};
```

### Submitting a Ticket with AI Classification
```javascript
// Frontend ticket submission
import { classifyTicket } from '../services/mlService';
import API from '../services/api';

const submitTicket = async (ticketData) => {
  try {
    // Get AI classification
    const classification = await classifyTicket(
      ticketData.title,
      ticketData.description
    );
    
    // Submit ticket with AI-enhanced data
    const response = await API.post('/api/tickets', {
      ...ticketData,
      category: classification.category,
      priority: classification.priority,
      aiConfidence: classification.confidence
    });
    
    return response.data;
  } catch (error) {
    console.error('Ticket submission failed:', error);
    throw error;
  }
};
```

### Real-time Task Tracking
```javascript
// Timer component for task tracking
import React, { useState, useEffect } from 'react';

const TaskTimer = ({ taskId }) => {
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
  
  const saveTimeLog = async () => {
    try {
      await API.post(`/api/tasks/${taskId}/time-log`, {
        duration: seconds,
        timestamp: new Date()
      });
      setSeconds(0);
      setIsRunning(false);
    } catch (error) {
      console.error('Failed to save time log:', error);
    }
  };
  
  return (
    <div className="task-timer">
      <div>{new Date(seconds * 1000).toISOString().substr(11, 8)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTimeLog}>Save</button>
    </div>
  );
};
```

## Troubleshooting

### JWT Token Issues
```javascript
// Check token validity
const verifyToken = () => {
  const token = localStorage.getItem('token');
  if (!token) return false;
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const isExpired = payload.exp * 1000 < Date.now();
    
    if (isExpired) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      return false;
    }
    return true;
  } catch (error) {
    return false;
  }
};
```

### MongoDB Connection Issues
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding
```bash
# Check ML service health
curl http://localhost:8000/health

# View ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization
```javascript
// Add MongoDB indexes for better query performance
// backend/models/Task.js
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ createdAt: -1 });
taskSchema.index({ dueDate: 1 });

// Use pagination for large datasets
exports.getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;
  
  const tasks = await Task.find()
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });
  
  const total = await Task.countDocuments();
  
  res.json({
    tasks,
    currentPage: page,
    totalPages: Math.ceil(total / limit),
    totalTasks: total
  });
};
```

This skill provides comprehensive guidance for implementing and extending the Enterprise User Management System with AI Analytics, covering authentication, task management, AI integration, and common development patterns.
