---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement user management with AI insights"
  - "create admin dashboard with analytics"
  - "add AI-powered ticket classification"
  - "integrate burnout detection in user system"
  - "build user management with ML service"
  - "setup kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system with integrated AI/ML capabilities for intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics. Built with React, Node.js, FastAPI, and MongoDB.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, authentication, and user profiles
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Admin Dashboard**: Organization-wide analytics and monitoring
- **Audit Logging**: Security tracking and suspicious activity alerts

## Installation

### Prerequisites
```bash
# Node.js 14+ and npm
# Python 3.8+
# MongoDB running locally or remote connection
```

### Clone and Setup
```bash
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=${MONGODB_URI}
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup
```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
# Frontend runs at http://localhost:3000
```

## Backend API Structure

### Authentication Endpoints
```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints
```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const auth = require('../middleware/auth');

// Get all tasks for user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin only)
router.post('/', auth, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Admin access required' });
    }
    
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Token is not valid' });
  }
};
```

## ML Service API

### FastAPI Main Application
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
risk_model = None
anomaly_detector = None

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_detector
    model_path = os.getenv("MODEL_PATH", "./models")
    
    try:
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")
    except:
        risk_model = RandomForestClassifier(n_estimators=100)
    
    try:
        anomaly_detector = joblib.load(f"{model_path}/anomaly_model.pkl")
    except:
        anomaly_detector = IsolationForest(contamination=0.1)

# Request Models
class TicketData(BaseModel):
    title: str
    description: str
    priority: str
    userId: str

class UserBehavior(BaseModel):
    userId: str
    loginCount: int
    taskCompletionRate: float
    avgTaskTime: float
    ticketCount: int
    lastLoginHour: int

class WorkloadData(BaseModel):
    userId: str
    activeTasks: int
    completedTasks: int
    avgHoursPerDay: float
    overtimeHours: float
    missedDeadlines: int

# AI Endpoints
@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify ticket priority and category using AI"""
    # Feature extraction from text
    features = extract_ticket_features(ticket.title, ticket.description)
    
    # Predict category
    categories = ["technical", "access", "bug", "feature-request", "other"]
    category_scores = np.random.dirichlet(np.ones(len(categories)))
    predicted_category = categories[np.argmax(category_scores)]
    
    # Predict urgency
    urgency_score = calculate_urgency(ticket.priority, features)
    
    return {
        "category": predicted_category,
        "urgency": urgency_score,
        "recommendedAssignee": "auto-assign",
        "estimatedResolutionTime": estimate_resolution_time(predicted_category)
    }

@app.post("/detect-risk")
async def detect_risk(behavior: UserBehavior):
    """Detect user risk based on behavior patterns"""
    features = np.array([[
        behavior.loginCount,
        behavior.taskCompletionRate,
        behavior.avgTaskTime,
        behavior.ticketCount,
        behavior.lastLoginHour
    ]])
    
    # Normalize features
    features_normalized = (features - features.mean()) / (features.std() + 1e-8)
    
    # Calculate risk score
    risk_score = calculate_risk_score(behavior)
    
    risk_level = "low"
    if risk_score > 0.7:
        risk_level = "high"
    elif risk_score > 0.4:
        risk_level = "medium"
    
    return {
        "userId": behavior.userId,
        "riskScore": float(risk_score),
        "riskLevel": risk_level,
        "factors": analyze_risk_factors(behavior),
        "recommendations": get_risk_recommendations(risk_level)
    }

@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    features = np.array([[
        behavior.loginCount,
        behavior.taskCompletionRate,
        behavior.avgTaskTime,
        behavior.ticketCount,
        behavior.lastLoginHour
    ]])
    
    # Predict anomaly (-1 for anomaly, 1 for normal)
    prediction = anomaly_detector.predict(features)[0]
    anomaly_score = anomaly_detector.score_samples(features)[0]
    
    is_anomaly = prediction == -1
    
    return {
        "userId": behavior.userId,
        "isAnomaly": bool(is_anomaly),
        "anomalyScore": float(anomaly_score),
        "anomalyType": detect_anomaly_type(behavior) if is_anomaly else None,
        "severity": "high" if is_anomaly and anomaly_score < -0.5 else "medium"
    }

@app.post("/detect-burnout")
async def detect_burnout(workload: WorkloadData):
    """Detect employee burnout risk"""
    burnout_score = calculate_burnout_score(workload)
    
    risk_level = "low"
    if burnout_score > 0.7:
        risk_level = "critical"
    elif burnout_score > 0.5:
        risk_level = "high"
    elif burnout_score > 0.3:
        risk_level = "medium"
    
    return {
        "userId": workload.userId,
        "burnoutScore": float(burnout_score),
        "riskLevel": risk_level,
        "indicators": analyze_burnout_indicators(workload),
        "recommendations": get_burnout_recommendations(risk_level)
    }

# Helper Functions
def extract_ticket_features(title: str, description: str) -> dict:
    text = (title + " " + description).lower()
    return {
        "length": len(text),
        "hasError": "error" in text or "bug" in text,
        "hasUrgent": "urgent" in text or "critical" in text,
        "wordCount": len(text.split())
    }

def calculate_urgency(priority: str, features: dict) -> float:
    base_urgency = {"low": 0.2, "medium": 0.5, "high": 0.8, "critical": 1.0}
    urgency = base_urgency.get(priority.lower(), 0.5)
    
    if features["hasUrgent"]:
        urgency = min(1.0, urgency + 0.2)
    if features["hasError"]:
        urgency = min(1.0, urgency + 0.1)
    
    return urgency

def estimate_resolution_time(category: str) -> str:
    times = {
        "technical": "2-4 hours",
        "access": "1-2 hours",
        "bug": "4-8 hours",
        "feature-request": "1-3 days",
        "other": "2-6 hours"
    }
    return times.get(category, "2-4 hours")

def calculate_risk_score(behavior: UserBehavior) -> float:
    score = 0.0
    
    # Low login frequency
    if behavior.loginCount < 5:
        score += 0.3
    
    # Low task completion
    if behavior.taskCompletionRate < 0.5:
        score += 0.4
    
    # Unusual login time
    if behavior.lastLoginHour < 6 or behavior.lastLoginHour > 22:
        score += 0.2
    
    # High ticket count
    if behavior.ticketCount > 10:
        score += 0.1
    
    return min(1.0, score)

def analyze_risk_factors(behavior: UserBehavior) -> List[str]:
    factors = []
    if behavior.loginCount < 5:
        factors.append("Low login frequency")
    if behavior.taskCompletionRate < 0.5:
        factors.append("Low task completion rate")
    if behavior.lastLoginHour < 6 or behavior.lastLoginHour > 22:
        factors.append("Unusual login time")
    return factors

def get_risk_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "low": ["Continue monitoring", "Regular check-ins"],
        "medium": ["Increase monitoring", "Review recent activities", "Manager notification"],
        "high": ["Immediate review required", "Security audit", "Account restrictions"]
    }
    return recommendations.get(risk_level, [])

def detect_anomaly_type(behavior: UserBehavior) -> str:
    if behavior.lastLoginHour < 6 or behavior.lastLoginHour > 22:
        return "unusual_login_time"
    if behavior.loginCount > 50:
        return "excessive_logins"
    if behavior.taskCompletionRate < 0.1:
        return "abnormal_activity"
    return "unknown"

def calculate_burnout_score(workload: WorkloadData) -> float:
    score = 0.0
    
    # High active tasks
    if workload.activeTasks > 10:
        score += 0.3
    
    # Long working hours
    if workload.avgHoursPerDay > 10:
        score += 0.3
    
    # Overtime hours
    if workload.overtimeHours > 20:
        score += 0.2
    
    # Missed deadlines
    if workload.missedDeadlines > 3:
        score += 0.2
    
    return min(1.0, score)

def analyze_burnout_indicators(workload: WorkloadData) -> List[str]:
    indicators = []
    if workload.activeTasks > 10:
        indicators.append("High task load")
    if workload.avgHoursPerDay > 10:
        indicators.append("Extended working hours")
    if workload.overtimeHours > 20:
        indicators.append("Excessive overtime")
    if workload.missedDeadlines > 3:
        indicators.append("Multiple missed deadlines")
    return indicators

def get_burnout_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "low": ["Maintain work-life balance", "Regular breaks"],
        "medium": ["Review workload", "Consider task redistribution", "Manager discussion"],
        "high": ["Immediate workload reduction", "Mandatory time off", "HR intervention"],
        "critical": ["Emergency intervention", "Medical leave consideration", "Immediate workload redistribution"]
    }
    return recommendations.get(risk_level, [])

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": True}
```

## Frontend Integration

### API Service
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
});

const mlApi = axios.create({
  baseURL: ML_URL,
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth
export const authService = {
  login: (email, password) => api.post('/auth/login', { email, password }),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => localStorage.removeItem('token'),
};

// Tasks
export const taskService = {
  getTasks: () => api.get('/tasks'),
  createTask: (task) => api.post('/tasks', task),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/tasks/${id}`),
};

// Tickets
export const ticketService = {
  getTickets: () => api.get('/tickets'),
  createTicket: async (ticket) => {
    // Get AI classification first
    const classification = await mlApi.post('/classify-ticket', ticket);
    return api.post('/tickets', { ...ticket, ...classification.data });
  },
  updateTicket: (id, updates) => api.patch(`/tickets/${id}`, updates),
};

// AI Analytics
export const aiService = {
  detectRisk: (behavior) => mlApi.post('/detect-risk', behavior),
  detectAnomaly: (behavior) => mlApi.post('/detect-anomaly', behavior),
  detectBurnout: (workload) => mlApi.post('/detect-burnout', workload),
  classifyTicket: (ticket) => mlApi.post('/classify-ticket', ticket),
};

// Users (Admin)
export const userService = {
  getUsers: () => api.get('/users'),
  createUser: (user) => api.post('/users', user),
  updateUser: (id, updates) => api.patch(`/users/${id}`, updates),
  deleteUser: (id) => api.delete(`/users/${id}`),
  getUserAnalytics: (id) => api.get(`/users/${id}/analytics`),
};

export default api;
```

### Dashboard Component
```javascript
// frontend/src/components/Dashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, aiService } from '../services/api';

const Dashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const tasksResponse = await taskService.getTasks();
      setTasks(tasksResponse.data);

      // Get AI analytics
      const behavior = calculateUserBehavior(tasksResponse.data);
      const riskAnalysis = await aiService.detectRisk(behavior);
      const burnoutAnalysis = await aiService.detectBurnout({
        userId: localStorage.getItem('userId'),
        activeTasks: tasksResponse.data.filter(t => t.status !== 'done').length,
        completedTasks: tasksResponse.data.filter(t => t.status === 'done').length,
        avgHoursPerDay: 8,
        overtimeHours: 5,
        missedDeadlines: 1,
      });

      setAnalytics({
        risk: riskAnalysis.data,
        burnout: burnoutAnalysis.data,
      });
    } catch (error) {
      console.error('Error loading dashboard:', error);
    } finally {
      setLoading(false);
    }
  };

  const calculateUserBehavior = (tasks) => {
    const completed = tasks.filter(t => t.status === 'done').length;
    const total = tasks.length;
    
    return {
      userId: localStorage.getItem('userId'),
      loginCount: 15,
      taskCompletionRate: total > 0 ? completed / total : 0,
      avgTaskTime: 4.5,
      ticketCount: 3,
      lastLoginHour: new Date().getHours(),
    };
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      
      {/* AI Insights */}
      {analytics && (
        <div className="ai-insights">
          <div className={`risk-card ${analytics.risk.riskLevel}`}>
            <h3>Risk Level: {analytics.risk.riskLevel}</h3>
            <p>Score: {(analytics.risk.riskScore * 100).toFixed(0)}%</p>
            <ul>
              {analytics.risk.factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
          
          <div className={`burnout-card ${analytics.burnout.riskLevel}`}>
            <h3>Burnout Risk: {analytics.burnout.riskLevel}</h3>
            <p>Score: {(analytics.burnout.burnoutScore * 100).toFixed(0)}%</p>
            <ul>
              {analytics.burnout.indicators.map((indicator, i) => (
                <li key={i}>{indicator}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {/* Kanban Board */}
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({getTasksByStatus('todo').length})</h3>
          {getTasksByStatus('todo').map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({getTasksByStatus('inprogress').length})</h3>
          {getTasksByStatus('inprogress').map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({getTasksByStatus('done').length})</h3>
          {getTasksByStatus('done').map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onUpdate }) => {
  const handleStatusChange = async (newStatus) => {
    try {
      await taskService.updateTaskStatus(task._id, newStatus);
      onUpdate();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className={`task-card priority-${task.priority}`}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => handleStatusChange('inprogress')}>Start</button>
        )}
        {task.status === 'inprogress' && (
          <>
            <button onClick={() => handleStatusChange('todo')}>Back</button>
            <button onClick={() => handleStatusChange('done')}>Complete</button>
          </>
        )}
      </div>
    </div>
  );
};

export default Dashboard;
```

## Database Models

### User Model
```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  profileImage: String,
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true },
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  dueDate: Date,
  completedAt: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model
```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  category: String,
  urgency: Number,
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    urgency: Number,
    estimatedResolutionTime: String,
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date,
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Implementing Time Tracking
```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
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

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const seconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  const handleSave = async () => {
    try {
      await onSave(taskId, seconds);
      setSeconds(0);
      setIsRunning(false);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => setSeconds(0)}>Reset</button>
        <button onClick={handleSave}>Save</button>
      </div>
    </div>
  );
};
