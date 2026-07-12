---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with task tracking"
  - "implement AI-powered ticket classification system"
  - "build admin dashboard for user and task management"
  - "add burnout detection and risk prediction features"
  - "configure kanban board with time tracking"
  - "integrate ML service for anomaly detection"
  - "deploy user management system with AI insights"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build and customize a full-stack enterprise user management system with integrated AI analytics. The system provides user/admin dashboards, task management with Kanban boards, support ticket systems, and ML-powered insights including risk detection, burnout analysis, and anomaly detection.

## What This Project Does

The Enterprise User Management System is a three-tier application:
- **Frontend (React)**: User/admin dashboards, Kanban boards, ticket management
- **Backend (Node.js)**: REST APIs, JWT authentication, business logic
- **ML Service (FastAPI)**: AI analytics including ticket classification, risk prediction, anomaly detection, and burnout analysis

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
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

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
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

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
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
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key Architecture Components

### Backend API Structure

**Authentication Routes** (`backend/routes/auth.js`):
```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

// User registration
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// User login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Middleware for Authentication** (`backend/middleware/auth.js`):
```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

**Task Management Routes** (`backend/routes/tasks.js`):
```javascript
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

const router = express.Router();

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task (admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.userId
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (task.assignedTo.toString() !== req.user.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Unauthorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
});

module.exports = router;
```

### ML Service API

**FastAPI ML Service** (`ml-service/main.py`):
```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
models = {}

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class UserActivityData(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_completion_time: float
    hours_worked: float
    tickets_raised: int

class RiskPredictionResponse(BaseModel):
    risk_level: str
    risk_score: float
    factors: List[str]

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models on startup"""
    model_path = os.getenv("MODEL_PATH", "./models")
    os.makedirs(model_path, exist_ok=True)
    
    # Initialize ticket classifier
    try:
        models['ticket_classifier'] = joblib.load(f"{model_path}/ticket_classifier.pkl")
    except:
        # Initialize with dummy model if not exists
        models['ticket_classifier'] = RandomForestClassifier(n_estimators=100)
        models['ticket_scaler'] = StandardScaler()

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and route to appropriate team"""
    try:
        # Simple rule-based classification (can be replaced with trained model)
        text = f"{ticket.title} {ticket.description}".lower()
        
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            department = 'engineering'
        elif any(word in text for word in ['password', 'access', 'login', 'permission']):
            category = 'access'
            department = 'it_support'
        elif any(word in text for word in ['payment', 'invoice', 'billing']):
            category = 'billing'
            department = 'finance'
        else:
            category = 'general'
            department = 'support'
        
        # Auto-assign priority based on keywords
        if any(word in text for word in ['urgent', 'critical', 'asap', 'emergency']):
            priority = 'high'
        elif ticket.priority:
            priority = ticket.priority
        else:
            priority = 'medium'
        
        return {
            "category": category,
            "department": department,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(activity: UserActivityData):
    """Predict user risk based on activity patterns"""
    try:
        risk_factors = []
        risk_score = 0.0
        
        # Burnout detection
        if activity.hours_worked > 50:
            risk_score += 0.3
            risk_factors.append("Excessive working hours")
        
        # Task completion ratio
        total_tasks = activity.tasks_completed + activity.tasks_pending
        if total_tasks > 0:
            completion_ratio = activity.tasks_completed / total_tasks
            if completion_ratio < 0.5:
                risk_score += 0.2
                risk_factors.append("Low task completion rate")
        
        # Average completion time anomaly
        if activity.avg_completion_time > 72:  # hours
            risk_score += 0.15
            risk_factors.append("Slow task completion")
        
        # High ticket volume
        if activity.tickets_raised > 10:
            risk_score += 0.2
            risk_factors.append("High support ticket volume")
        
        # Pending task overload
        if activity.tasks_pending > 15:
            risk_score += 0.15
            risk_factors.append("Task overload")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPredictionResponse(
            risk_level=risk_level,
            risk_score=min(risk_score, 1.0),
            factors=risk_factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(activity: UserActivityData):
    """Detect anomalous user behavior"""
    try:
        anomalies = []
        
        # Unusual working hours
        if activity.hours_worked > 80 or activity.hours_worked < 5:
            anomalies.append({
                "type": "working_hours",
                "severity": "high" if activity.hours_worked > 80 else "medium",
                "message": f"Unusual working hours: {activity.hours_worked}h"
            })
        
        # Sudden spike in tickets
        if activity.tickets_raised > 20:
            anomalies.append({
                "type": "ticket_spike",
                "severity": "medium",
                "message": f"High ticket volume: {activity.tickets_raised}"
            })
        
        # Zero productivity
        if activity.tasks_completed == 0 and activity.tasks_pending > 5:
            anomalies.append({
                "type": "low_productivity",
                "severity": "high",
                "message": "No tasks completed with pending workload"
            })
        
        return {
            "is_anomalous": len(anomalies) > 0,
            "anomalies": anomalies,
            "timestamp": str(np.datetime64('now'))
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(activity: UserActivityData):
    """Analyze burnout risk for user"""
    try:
        burnout_score = 0.0
        indicators = []
        
        # Working hours
        if activity.hours_worked > 60:
            burnout_score += 0.4
            indicators.append("Excessive working hours")
        elif activity.hours_worked > 50:
            burnout_score += 0.2
            indicators.append("High working hours")
        
        # Task overload
        if activity.tasks_pending > 20:
            burnout_score += 0.3
            indicators.append("Severe task overload")
        elif activity.tasks_pending > 10:
            burnout_score += 0.15
            indicators.append("Task overload")
        
        # Slow completion
        if activity.avg_completion_time > 96:
            burnout_score += 0.2
            indicators.append("Very slow task completion")
        
        # Support dependency
        if activity.tickets_raised > 15:
            burnout_score += 0.1
            indicators.append("High support dependency")
        
        burnout_level = "high" if burnout_score >= 0.7 else "medium" if burnout_score >= 0.4 else "low"
        
        recommendations = []
        if burnout_score >= 0.4:
            recommendations.append("Reduce working hours to recommended levels")
            recommendations.append("Redistribute tasks to balance workload")
            recommendations.append("Schedule one-on-one with manager")
        
        return {
            "burnout_level": burnout_level,
            "burnout_score": min(burnout_score, 1.0),
            "indicators": indicators,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### Frontend React Components

**API Service** (`frontend/src/services/api.js`):
```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instances
const api = axios.create({
  baseURL: API_URL,
});

const mlApi = axios.create({
  baseURL: ML_API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me'),
};

// Task API
export const taskAPI = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`),
};

// Ticket API
export const ticketAPI = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  updateTicket: (id, updates) => api.patch(`/api/tickets/${id}`, updates),
  classifyTicket: (ticketData) => mlApi.post('/api/ml/classify-ticket', ticketData),
};

// ML Analytics API
export const mlAPI = {
  predictRisk: (activityData) => mlApi.post('/api/ml/predict-risk', activityData),
  detectAnomaly: (activityData) => mlApi.post('/api/ml/detect-anomaly', activityData),
  analyzeBurnout: (activityData) => mlApi.post('/api/ml/burnout-analysis', activityData),
};

// User Management API (Admin)
export const userAPI = {
  getUsers: () => api.get('/api/users'),
  createUser: (userData) => api.post('/api/users', userData),
  updateUser: (id, updates) => api.patch(`/api/users/${id}`, updates),
  deleteUser: (id) => api.delete(`/api/users/${id}`),
};

export default api;
```

**Kanban Board Component** (`frontend/src/components/KanbanBoard.jsx`):
```javascript
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done'),
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
    e.dataTransfer.setData('currentStatus', task.status);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.dueDate && (
          <span className="due-date">
            Due: {new Date(task.dueDate).toLocaleDateString()}
          </span>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'in_progress')}
        onDragOver={handleDragOver}
      >
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'done')}
        onDragOver={handleDragOver}
      >
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

**AI Analytics Dashboard** (`frontend/src/components/AIAnalytics.jsx`):
```javascript
import React, { useState, useEffect } from 'react';
import { mlAPI } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = ({ userId, activityData }) => {
  const [riskAnalysis, setRiskAnalysis] = useState(null);
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);
  const [anomalies, setAnomalies] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (activityData) {
      performAnalysis();
    }
  }, [activityData]);

  const performAnalysis = async () => {
    try {
      setLoading(true);
      
      const [risk, burnout, anomaly] = await Promise.all([
        mlAPI.predictRisk(activityData),
        mlAPI.analyzeBurnout(activityData),
        mlAPI.detectAnomaly(activityData),
      ]);
      
      setRiskAnalysis(risk.data);
      setBurnoutAnalysis(burnout.data);
      setAnomalies(anomaly.data);
    } catch (error) {
      console.error('Error performing AI analysis:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Analyzing data...</div>;

  return (
    <div className="ai-analytics-container">
      <h2>AI-Powered Insights</h2>
      
      {/* Risk Analysis */}
      <div className={`analysis-card risk-${riskAnalysis?.risk_level}`}>
        <h3>Risk Assessment</h3>
        <div className="risk-score">
          Risk Level: <strong>{riskAnalysis?.risk_level.toUpperCase()}</strong>
        </div>
        <div className="score-bar">
          <div
            className="score-fill"
            style={{ width: `${(riskAnalysis?.risk_score || 0) * 100}%` }}
          />
        </div>
        <ul className="factors-list">
          {riskAnalysis?.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      {/* Burnout Analysis */}
      <div className={`analysis-card burnout-${burnoutAnalysis?.burnout_level}`}>
        <h3>Burnout Analysis</h3>
        <div className="burnout-level">
          Level: <strong>{burnoutAnalysis?.burnout_level.toUpperCase()}</strong>
        </div>
        <div className="indicators">
          <h4>Indicators:</h4>
          <ul>
            {burnoutAnalysis?.indicators.map((indicator, idx) => (
              <li key={idx}>{indicator}</li>
            ))}
          </ul>
        </div>
        {burnoutAnalysis?.recommendations.length > 0 && (
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {burnoutAnalysis.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        )}
      </div>

      {/* Anomaly Detection */}
      {anomalies?.is_anomalous && (
        <div className="analysis-card anomaly-alert">
          <h3>⚠️ Anomalies Detected</h3>
          <ul>
            {anomalies.anomalies.map((anomaly, idx) => (
              <li key={idx} className={`severity-${anomaly.severity}`}>
                <strong>{anomaly.type}:</strong> {anomaly.message}
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

**User Model** (`backend/models/User.js`):
```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user',
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active',
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
  lastLogin: Date,
});

module.exports = mongoose.model('User', userSchema);
```

**Task Model** (`backend/models/Task.js`):
```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo',
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium',
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  dueDate: Date,
  completedAt: Date,
  createdAt: {
    type: Date,
    default: Date.now,
  },
  timeTracked: {
    type: Number,
    default: 0, // in seconds
  },
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Adding New ML Analytics Feature

1. Define endpoint in ML service:
```python
@app.post("/api/ml/my-new-analysis")
async def my_new_analysis(data: MyDataModel):
    # Implement ML logic
    result = perform_analysis(data)
    return result
```

2. Add to frontend API service:
```javascript
export const mlAPI = {
  myNewAnalysis: (data) => mlApi.post('/api/ml/my-new-analysis', data),
};
```

3. Use in React component:
```javascript
const result = await mlAPI.myNewAnalysis(inputData);
setAnalysisResult(result.data);
```

### Implementing Role-Based Access

```javascript
// Backend route protection
router.get('/admin/users', authMiddleware, adminMiddleware, async (req, res) => {
  // Admin-only logic
});

// Frontend component protection
import { useAuth } from '../context/AuthContext';

const AdminPanel = () => {
  const { user } = useAuth();
  
  if (user?.role !== 'admin') {
    return <div>Access Denied</div>;
  }
  
  return <div>Admin Content</div>;
};
```

### Time Tracking Implementation

```javascript
// Frontend stopwatch hook
const useStopwatch = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  
