---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task tracking capabilities
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user management system with task tracking"
  - "integrate AI-based risk detection and anomaly detection"
  - "build admin dashboard with user analytics"
  - "create kanban board with time tracking"
  - "add AI ticket classification system"
  - "configure JWT authentication for user management"
  - "deploy enterprise management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack application for managing users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, user CRUD operations
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, alerts
- **Authentication**: JWT-based secure login

## Installation

### Full Stack Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**
```env
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=info
```

## Running the Application

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Create user
POST /api/users
Headers: { Authorization: "Bearer <token>" }
{
  "username": "jane_smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer <token>" }
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { Authorization: "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "priority": "high",
  "status": "todo",
  "assignedTo": "user_id",
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:id
Headers: { Authorization: "Bearer <token>" }
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}

// Track time
POST /api/tasks/:id/time
Headers: { Authorization: "Bearer <token>" }
{
  "duration": 45 // minutes
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Login issue",
  "description": "Cannot access account",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Headers: { Authorization: "Bearer <token>" }

// Update ticket
PUT /api/tickets/:id
Headers: { Authorization: "Bearer <token>" }
{
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
Headers: { Authorization: "Bearer <token>" }
{
  "userId": "user_id",
  "behavior": {
    "loginAttempts": 5,
    "failedLogins": 3,
    "unusualActivity": true,
    "accessPatterns": [...]
  }
}
// Returns: { riskScore: 0.85, riskLevel: "high" }

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "activityData": {
    "loginTime": "2026-04-15T03:00:00Z",
    "location": "Unknown",
    "deviceType": "New"
  }
}
// Returns: { isAnomaly: true, confidence: 0.92 }

// Burnout detection
POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "workloadData": {
    "tasksCompleted": 45,
    "overtimeHours": 30,
    "weekendWork": 8,
    "taskCompletionRate": 0.6
  }
}
// Returns: { burnoutScore: 0.78, recommendation: "Reduce workload" }

// Project delay prediction
POST /api/ml/predict-delay
{
  "projectId": "project_id",
  "metrics": {
    "completionRate": 0.45,
    "daysRemaining": 10,
    "teamSize": 5,
    "complexity": "high"
  }
}
// Returns: { delayProbability: 0.82, estimatedDelay: 5 }

// Ticket classification
POST /api/ml/classify-ticket
{
  "title": "Cannot login to system",
  "description": "Getting authentication error"
}
// Returns: { category: "technical", priority: "high", suggestedAgent: "IT Support" }
```

## Frontend Integration Examples

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      const { token, user } = response.data;
      localStorage.setItem('token', token);
      setToken(token);
      setUser(user);
      return { success: true };
    } catch (error) {
      return { success: false, error: error.response?.data?.message };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout };
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus} 
          />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus} 
          />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onStatusChange={updateTaskStatus} 
          />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      
      // Parallel requests for different analytics
      const [riskData, burnoutData] = await Promise.all([
        axios.post(
          `${ML_API_URL}/api/ml/predict-risk`,
          { userId },
          { headers: { Authorization: `Bearer ${token}` } }
        ),
        axios.post(
          `${ML_API_URL}/api/ml/detect-burnout`,
          { userId },
          { headers: { Authorization: `Bearer ${token}` } }
        )
      ]);

      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h4>Risk Level</h4>
        <p className={`risk-${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </p>
        <span>Score: {analytics.risk.riskScore.toFixed(2)}</span>
      </div>
      
      <div className="metric-card">
        <h4>Burnout Detection</h4>
        <p>Score: {analytics.burnout.burnoutScore.toFixed(2)}</p>
        <span>{analytics.burnout.recommendation}</span>
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

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({ username, email, password, role });
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({ token, user: user.toJSON() });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const user = await User.findByIdAndUpdate(id, updates, { new: true })
      .select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
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
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      assignedTo,
      createdBy: req.userId,
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.userId })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const task = await Task.findByIdAndUpdate(id, updates, { new: true });
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { id } = req.params;
    const { duration } = req.body;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI()

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    userId: str
    behavior: dict

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityData: dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadData: dict

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        behavior = request.behavior
        
        # Extract features
        login_attempts = behavior.get('loginAttempts', 0)
        failed_logins = behavior.get('failedLogins', 0)
        unusual_activity = 1 if behavior.get('unusualActivity', False) else 0
        
        # Simple risk scoring algorithm
        risk_score = (
            (failed_logins / max(login_attempts, 1)) * 0.4 +
            unusual_activity * 0.6
        )
        
        risk_level = 'low'
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        
        return {
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        activity = request.activityData
        
        # Simple anomaly detection logic
        is_anomaly = False
        confidence = 0.0
        
        # Check for unusual login time (e.g., 2-5 AM)
        login_time = activity.get('loginTime', '')
        if '02:' in login_time or '03:' in login_time or '04:' in login_time:
            is_anomaly = True
            confidence = 0.8
        
        # Check for unknown location
        if activity.get('location') == 'Unknown':
            is_anomaly = True
            confidence = max(confidence, 0.7)
        
        # Check for new device
        if activity.get('deviceType') == 'New':
            is_anomaly = True
            confidence = max(confidence, 0.65)
        
        return {
            "isAnomaly": is_anomaly,
            "confidence": round(confidence, 2)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        workload = request.workloadData
        
        # Extract workload metrics
        overtime_hours = workload.get('overtimeHours', 0)
        weekend_work = workload.get('weekendWork', 0)
        completion_rate = workload.get('taskCompletionRate', 1.0)
        
        # Burnout scoring
        burnout_score = (
            (overtime_hours / 40) * 0.3 +
            (weekend_work / 16) * 0.3 +
            (1 - completion_rate) * 0.4
        )
        
        recommendation = "Workload is healthy"
        if burnout_score > 0.7:
            recommendation = "High burnout risk - Reduce workload immediately"
        elif burnout_score > 0.5:
            recommendation = "Moderate burnout risk - Consider reducing workload"
        
        return {
            "burnoutScore": round(min(burnout_score, 1.0), 2),
            "recommendation": recommendation
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-delay")
async def predict_delay(request: dict):
    try:
        metrics = request.get('metrics', {})
        
        completion_rate = metrics.get('completionRate', 0.5)
        days_remaining = metrics.get('daysRemaining', 10)
        complexity = metrics.get('complexity', 'medium')
        
        # Simple delay prediction
        complexity_factor = {'low': 0.5, 'medium': 1.0, 'high': 1.5}
        factor = complexity_factor.get(complexity, 1.0)
        
        delay_probability = (1 - completion_rate) * factor
        estimated_delay = int((1 - completion_rate) * days_remaining)
        
        return {
            "delayProbability": round(min(delay_probability, 1.0), 2),
            "estimatedDelay": estimated_delay
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        title = request.title.lower()
        description = request.description.lower()
        
        # Simple keyword-based classification
        category = "general"
        priority = "low"
        suggested_agent = "General Support"
        
        # Technical issues
        if any(word in title + description for word in ['login', 'password', 'authentication', 'access']):
            category = "technical"
            priority = "high"
            suggested_agent = "IT Support"
        
        # Account issues
        elif any(word in title + description for word in ['account', 'profile', 'settings']):
            category = "account"
            priority = "medium"
            suggested_agent = "Account Support"
        
        # Billing issues
        elif any(word in title + description for word in ['billing', 'payment', 'invoice']):
            category = "billing"
            priority = "high"
            suggested_agent = "Billing Support"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAgent": suggested_agent
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Database Models

### User Model

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
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

userSchema.methods.toJSON = function() {
  const obj = this.toObject();
  delete obj.password;
  return obj;
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    trim: true
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'account', 'billing', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  resolution: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

ticketSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### API Request Helper

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const getAuthHeaders = () => ({
  Authorization: `Bearer ${localStorage.getItem('token')}`
});

export const api = {
  // Auth
  login: (email, password) =>
    axios.post(`${API_URL}/api/auth/login`, { email, password }),
  
  register: (userData) =>
    axios.post(`${API_URL}/api/auth/register`, userData),
  
  // Users
  getUsers: () =>
    axios.get(`${API_URL}/api/users`, { headers: getAuthHeaders() }),
  
  createUser: (userData) =>
    axios.post(`${API_URL}/api/users`, userData, { headers: getAuthHeaders() }),
  
  updateUser: (id, updates) =>
    axios.put(`${API_URL}/api/users/${id}`, updates, { headers: getAuthHeaders() }),
  
  deleteUser: (id) =>
    axios.delete(`${API_URL}/api/users/${id}`, { headers: getAuthHeaders() }),
  
  // Tasks
  getTasks: () =>
    axios.get(`${API_URL}/api/tasks`, { headers: getAuthHeaders() }),
  
  createTask: (taskData) =>
    axios.post(`${API_URL}/api/tasks`, taskData, { headers: getAuthHeaders() }),
  
  updateTask: (id, updates) =>
    axios.put(`${API_URL}/api/tasks/${id}`, updates, { headers: getAuthHeaders() }),
  
  trackTime: (id, duration) =>
    axios.post(`${API_URL}/api/tasks/${id}/time`, { duration }, { headers: getAuthHeaders() }),
  
  // Tickets
  getTickets: () =>
    axios.get(`${API
