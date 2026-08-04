---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with ML insights"
  - "implement AI-powered ticket classification system"
  - "build task tracking with burnout detection"
  - "add anomaly detection to user management"
  - "integrate AI analytics into admin dashboard"
  - "develop kanban board with time tracking"
  - "create role-based user management with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack user management platform that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React frontend, Node.js/Spring Boot backend, FastAPI ML service, and MongoDB.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Tracking**: Kanban-style boards with time tracking and status management
- **Ticket System**: Support ticket creation, assignment, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Organization-wide metrics, audit logs, and security alerts

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB
- npm or yarn

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
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management APIs

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Get user by ID
GET /api/users/:id
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer <token>" }
{
  "name": "Updated Name",
  "role": "admin",
  "department": "Engineering"
}

// Delete user (Admin only)
DELETE /api/users/:id
Headers: { Authorization: "Bearer <token>" }
```

### Task Management APIs

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo", // todo, in-progress, done
  "dueDate": "2026-05-01"
}

// Get tasks for user
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }

// Update task status
PUT /api/tasks/:taskId
Headers: { Authorization: "Bearer <token>" }
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}
```

### Ticket APIs

```javascript
// Create support ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/user/:userId
Headers: { Authorization: "Bearer <token>" }

// Update ticket (Admin)
PUT /api/tickets/:ticketId
Headers: { Authorization: "Bearer <token>" }
{
  "status": "resolved",
  "assignedTo": "admin_user_id",
  "resolution": "Fixed permission issue"
}
```

### AI Analytics APIs

```javascript
// Classify ticket using AI
POST /api/ml/classify-ticket
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Login page not loading",
  "description": "Browser shows timeout error"
}
// Returns: { category: "technical", priority: "high", suggestedAssignee: "user_id" }

// Predict user risk
POST /api/ml/predict-risk
Headers: { Authorization: "Bearer <token>" }
{
  "userId": "user_id"
}
// Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }

// Detect burnout
POST /api/ml/burnout-detection
Headers: { Authorization: "Bearer <token>" }
{
  "userId": "user_id"
}
// Returns: { burnoutScore: 0.82, recommendation: "Reduce workload" }

// Predict project delays
POST /api/ml/predict-delay
Headers: { Authorization: "Bearer <token>" }
{
  "projectId": "project_id",
  "tasksCompleted": 15,
  "totalTasks": 50,
  "averageVelocity": 3.2
}
// Returns: { delayProbability: 0.65, estimatedDelay: 5 }
```

## Frontend Implementation Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (err) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    setToken(res.data.token);
    setUser(res.data.user);
    localStorage.setItem('token', res.data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (err) {
      console.error('Error fetching tasks:', err);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (err) {
      console.error('Error updating task:', err);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      <div className="kanban-column" onDrop={(e) => handleDrop(e, 'todo')} onDragOver={handleDragOver}>
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => handleDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
      <div className="kanban-column" onDrop={(e) => handleDrop(e, 'in-progress')} onDragOver={handleDragOver}>
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => handleDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
      <div className="kanban-column" onDrop={(e) => handleDrop(e, 'done')} onDragOver={handleDragOver}>
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card" draggable onDragStart={(e) => handleDragStart(e, task._id)}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      const [risk, burnout] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/predict-risk`, { userId }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/burnout-detection`, { userId })
      ]);
      setRiskData(risk.data);
      setBurnoutData(burnout.data);
    } catch (err) {
      console.error('Error fetching analytics:', err);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-score ${riskData.riskLevel}`}>
          {(riskData.riskScore * 100).toFixed(1)}%
        </div>
        <p>Risk Level: {riskData.riskLevel}</p>
        <ul>
          {riskData.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>
      <div className="metric-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-score ${burnoutData.burnoutScore > 0.7 ? 'high' : 'normal'}`}>
          {(burnoutData.burnoutScore * 100).toFixed(1)}%
        </div>
        <p>{burnoutData.recommendation}</p>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### User Controller (Node.js)

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register user
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = await User.create({ name, email, password, role });
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });

    res.status(201).json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }

    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const { name, role, department } = req.body;

    if (req.user.role !== 'admin' && req.user.id !== id) {
      return res.status(403).json({ message: 'Access denied' });
    }

    const user = await User.findByIdAndUpdate(
      id,
      { name, role, department },
      { new: true }
    ).select('-password');

    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, status, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority,
      status,
      dueDate
    });

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { taskId } = req.params;
    const updates = req.body;

    const task = await Task.findByIdAndUpdate(taskId, updates, { new: true });
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation (Python/FastAPI)

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional
import os

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskRequest(BaseModel):
    userId: str

class BurnoutRequest(BaseModel):
    userId: str

class DelayRequest(BaseModel):
    projectId: str
    tasksCompleted: int
    totalTasks: int
    averageVelocity: float

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """AI-based ticket classification"""
    try:
        # Simple rule-based classification (replace with ML model)
        text = f"{request.title} {request.description}".lower()
        
        if any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ["slow", "performance", "timeout"]):
            category = "performance"
            priority = "medium"
        elif any(word in text for word in ["password", "login", "access"]):
            category = "security"
            priority = "high"
        else:
            category = "general"
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": None,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Mock risk calculation (replace with actual ML model)
        risk_score = np.random.uniform(0.2, 0.9)
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        factors = []
        if risk_score > 0.7:
            factors.append("High number of failed login attempts")
            factors.append("Unusual access patterns detected")
        
        return {
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout based on workload"""
    try:
        # Mock burnout calculation
        burnout_score = np.random.uniform(0.3, 0.95)
        
        if burnout_score > 0.8:
            recommendation = "Critical: Immediate workload reduction recommended"
        elif burnout_score > 0.6:
            recommendation = "Warning: Consider redistributing tasks"
        else:
            recommendation = "Workload is within healthy range"
        
        return {
            "burnoutScore": float(burnout_score),
            "recommendation": recommendation,
            "workloadMetrics": {
                "weeklyHours": 45,
                "tasksInProgress": 8,
                "overdueTasks": 2
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-delay")
async def predict_delay(request: DelayRequest):
    """Predict project delay probability"""
    try:
        completion_rate = request.tasksCompleted / request.totalTasks
        expected_rate = 0.5  # 50% should be done at midpoint
        
        if completion_rate < expected_rate:
            delay_probability = 0.75
            estimated_delay = int((1 - completion_rate / expected_rate) * 10)
        else:
            delay_probability = 0.25
            estimated_delay = 0
        
        return {
            "delayProbability": delay_probability,
            "estimatedDelay": estimated_delay,
            "completionRate": completion_rate,
            "recommendation": "Increase team velocity" if delay_probability > 0.5 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Configuration

### MongoDB Connection

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token invalid' });
  }
};

exports.adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};
```

## Common Patterns

### Creating a Protected Route

```javascript
// backend/routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { protect, adminOnly } = require('../middleware/auth');
const {
  createTask,
  getUserTasks,
  updateTask,
  deleteTask
} = require('../controllers/taskController');

router.post('/', protect, createTask);
router.get('/user/:userId', protect, getUserTasks);
router.put('/:taskId', protect, updateTask);
router.delete('/:taskId', protect, adminOnly, deleteTask);

module.exports = router;
```

### Using AI Endpoints in Frontend

```javascript
// src/services/aiService.js
import axios from 'axios';

const ML_API = process.env.REACT_APP_ML_API_URL;

export const classifyTicket = async (title, description) => {
  const response = await axios.post(`${ML_API}/classify-ticket`, {
    title,
    description
  });
  return response.data;
};

export const getUserRisk = async (userId) => {
  const response = await axios.post(`${ML_API}/predict-risk`, { userId });
  return response.data;
};

export const checkBurnout = async (userId) => {
  const response = await axios.post(`${ML_API}/burnout-detection`, { userId });
  return response.data;
};

// Usage in component
import { classifyTicket } from '../services/aiService';

const createTicket = async (ticketData) => {
  const classification = await classifyTicket(ticketData.title, ticketData.description);
  
  const ticket = {
    ...ticketData,
    category: classification.category,
    priority: classification.priority
  };
  
  await axios.post(`${process.env.REACT_APP_API_URL}/tickets`, ticket);
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection string in .env
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
```

### JWT Authentication Errors

```javascript
// Ensure token is properly set in axios defaults
import axios from 'axios';

const token = localStorage.getItem('token');
if (token) {
  axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
}

// Handle expired tokens
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response && error.response.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
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

### ML Service Not Responding

```bash
# Check if FastAPI is running
curl http://localhost:8000/health

# View logs
uvicorn main:app --reload --log-level debug

# Install missing dependencies
pip install -r requirements.txt
```

### Frontend Build Errors

```bash
# Clear cache and reinstall
rm -rf node_modules package-lock.json
npm install

# Check environment variables
cat .env
# Should contain REACT_APP_API_URL and REACT_APP_ML_API_URL
```

### Database Schema Issues

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
});

module.exports = mongoose.model('User', userSchema);
```

## Performance Optimization

### Implement Caching

```javascript
// backend/middleware/cache.js
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

exports.cacheMiddleware = (duration) => (req, res, next) => {
  const key = req.originalUrl;
  const cachedResponse = cache.get(key);

  if (cachedResponse) {
    return res.json(cachedResponse);
  }

  res.originalJson = res.json;
  res.json = (data) => {
    cache.set(key, data, duration);
    res.originalJson(data);
  };
  next();
};
```

### Pagination for Large Datasets

```javascript
// backend/controllers/userController.js
exports.getAllUsers = async (req, res) => {
  try {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 10;
    const skip = (page - 1) * limit;

    const users = await User.find()
      .select('-password')
      .skip(skip)
      .limit(limit);

    const total = await User.countDocuments();

    res.json({
      users,
      currentPage: page,
      totalPages: Math.ceil(total / limit),
      total
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```
