---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered user management system"
  - "create user management dashboard with risk detection"
  - "build task management with burnout analysis"
  - "integrate AI ticket classification system"
  - "deploy user management with anomaly detection"
  - "configure AI analytics for user management"
  - "implement kanban board with AI insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-driven analytics. It provides:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban boards, time tracking, and assignment workflows
- **Support Tickets**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Centralized monitoring with audit logs and alerts

The system is built with React frontend, Node.js backend, MongoDB database, and FastAPI ML service using scikit-learn and River for online learning.

## Installation

### Prerequisites

```bash
# Required tools
node --version  # v14+ required
python --version  # Python 3.8+ required
mongo --version  # MongoDB 4.4+ required
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Configure .env with MongoDB URI and JWT secret
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
cp .env.example .env
# Configure .env with API endpoints
npm start  # Runs on http://localhost:3000
```

## Backend Configuration

### Environment Variables (backend/.env)

```bash
# MongoDB
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
# or MongoDB Atlas
# MONGODB_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/dbname

# JWT Authentication
JWT_SECRET=your_secure_random_secret_key_here
JWT_EXPIRE=7d

# Server
PORT=5000
NODE_ENV=development

# ML Service URL
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASS=${EMAIL_PASS}
```

### Key Backend API Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const user = new User({
      username,
      email,
      password,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate, authorize } = require('../middleware/auth');

// Get all tasks (with filters)
router.get('/', authenticate, async (req, res) => {
  try {
    const { status, assignedTo, priority } = req.query;
    const filter = {};
    
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user.id;
    }
    
    if (status) filter.status = status;
    if (assignedTo && req.user.role === 'admin') filter.assignedTo = assignedTo;
    if (priority) filter.priority = priority;
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin only)
router.post('/', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Configuration

### Environment Variables (ml-service/.env)

```bash
# ML Service
ML_HOST=0.0.0.0
ML_PORT=8000

# Model paths
MODEL_DIR=./models
DATA_DIR=./data

# Training settings
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.75
```

### AI Analytics API

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, preprocessing
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models
risk_model = None
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)
scaler = preprocessing.StandardScaler()

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    avg_task_time: float
    failed_login_attempts: int
    late_submissions: int
    active_hours: list

class TicketData(BaseModel):
    title: str
    description: str
    priority: str
    category: str = None

@app.on_event("startup")
async def load_models():
    global risk_model
    model_path = os.getenv("MODEL_DIR", "./models")
    
    if os.path.exists(f"{model_path}/risk_model.pkl"):
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")

@app.post("/api/ml/risk-prediction")
async def predict_risk(data: UserBehavior):
    """Predict user risk level based on behavior patterns"""
    try:
        features = np.array([[
            data.login_frequency,
            data.task_completion_rate,
            data.avg_task_time,
            data.failed_login_attempts,
            data.late_submissions,
            np.mean(data.active_hours) if data.active_hours else 0
        ]])
        
        if risk_model:
            risk_score = risk_model.predict_proba(features)[0][1]
        else:
            # Fallback heuristic
            risk_score = (
                (1 - data.task_completion_rate) * 0.3 +
                (data.failed_login_attempts / 10) * 0.3 +
                (data.late_submissions / 10) * 0.2 +
                (1 if data.login_frequency < 0.3 else 0) * 0.2
            )
        
        threshold = float(os.getenv("RISK_THRESHOLD", 0.7))
        
        return {
            "user_id": data.user_id,
            "risk_score": float(risk_score),
            "risk_level": "high" if risk_score > threshold else "medium" if risk_score > 0.4 else "low",
            "factors": {
                "task_completion": data.task_completion_rate,
                "failed_logins": data.failed_login_attempts,
                "late_submissions": data.late_submissions
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(data: UserBehavior):
    """Detect anomalous user behavior using online learning"""
    try:
        features = {
            'login_frequency': data.login_frequency,
            'task_completion_rate': data.task_completion_rate,
            'avg_task_time': data.avg_task_time,
            'failed_login_attempts': float(data.failed_login_attempts),
            'late_submissions': float(data.late_submissions)
        }
        
        # Scale features
        scaled_features = scaler.learn_one(features).transform_one(features)
        
        # Get anomaly score
        score = anomaly_detector.score_one(scaled_features)
        anomaly_detector.learn_one(scaled_features)
        
        threshold = float(os.getenv("ANOMALY_THRESHOLD", 0.8))
        
        return {
            "user_id": data.user_id,
            "anomaly_score": float(score),
            "is_anomaly": score > threshold,
            "detected_patterns": {
                "unusual_login": data.login_frequency < 0.2,
                "high_failures": data.failed_login_attempts > 5,
                "performance_drop": data.task_completion_rate < 0.5
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(data: UserBehavior):
    """Analyze potential employee burnout based on workload metrics"""
    try:
        # Calculate burnout indicators
        workload_score = min(data.avg_task_time / 8.0, 1.0)  # Normalized to 8 hours
        completion_stress = 1 - data.task_completion_rate
        time_pressure = data.late_submissions / 10.0
        
        active_hours_variance = np.std(data.active_hours) if len(data.active_hours) > 1 else 0
        irregular_hours = min(active_hours_variance / 4.0, 1.0)
        
        burnout_score = (
            workload_score * 0.35 +
            completion_stress * 0.25 +
            time_pressure * 0.25 +
            irregular_hours * 0.15
        )
        
        threshold = float(os.getenv("BURNOUT_THRESHOLD", 0.75))
        
        return {
            "user_id": data.user_id,
            "burnout_score": float(burnout_score),
            "risk_level": "critical" if burnout_score > threshold else "warning" if burnout_score > 0.5 else "normal",
            "indicators": {
                "workload": float(workload_score),
                "completion_stress": float(completion_stress),
                "time_pressure": float(time_pressure),
                "irregular_hours": float(irregular_hours)
            },
            "recommendations": [
                "Reduce task assignment" if workload_score > 0.7 else None,
                "Provide support" if completion_stress > 0.6 else None,
                "Adjust deadlines" if time_pressure > 0.5 else None,
                "Monitor work-life balance" if irregular_hours > 0.6 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/ticket-classification")
async def classify_ticket(ticket: TicketData):
    """Classify and route support tickets using NLP"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification (can be replaced with ML model)
        categories = {
            "technical": ["bug", "error", "crash", "not working", "broken", "issue"],
            "access": ["login", "password", "permission", "access", "locked"],
            "feature": ["feature", "request", "enhancement", "add", "improve"],
            "performance": ["slow", "lag", "performance", "timeout", "delay"],
            "data": ["data", "database", "record", "missing", "incorrect"]
        }
        
        scores = {}
        for category, keywords in categories.items():
            scores[category] = sum(1 for keyword in keywords if keyword in text)
        
        predicted_category = max(scores, key=scores.get) if max(scores.values()) > 0 else "general"
        
        # Priority suggestion
        urgent_keywords = ["urgent", "critical", "asap", "emergency", "down"]
        suggested_priority = "high" if any(kw in text for kw in urgent_keywords) else ticket.priority
        
        return {
            "category": predicted_category,
            "suggested_priority": suggested_priority,
            "confidence": max(scores.values()) / len(categories) if scores else 0.2,
            "routing": {
                "technical": "IT Support Team",
                "access": "Security Team",
                "feature": "Product Team",
                "performance": "DevOps Team",
                "data": "Data Team"
            }.get(predicted_category, "General Support")
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_BASE = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_BASE,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => api.post('/api/auth/login', { email, password }),
  register: (userData) => api.post('/api/auth/register', userData),
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

export const taskService = {
  getTasks: (filters) => api.get('/api/tasks', { params: filters }),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`)
};

export const ticketService = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  updateTicket: (id, data) => api.patch(`/api/tickets/${id}`, data)
};

export const mlService = {
  predictRisk: (behaviorData) => 
    axios.post(`${ML_BASE}/api/ml/risk-prediction`, behaviorData),
  
  detectAnomaly: (behaviorData) => 
    axios.post(`${ML_BASE}/api/ml/anomaly-detection`, behaviorData),
  
  analyzeBurnout: (behaviorData) => 
    axios.post(`${ML_BASE}/api/ml/burnout-analysis`, behaviorData),
  
  classifyTicket: (ticketData) => 
    axios.post(`${ML_BASE}/api/ml/ticket-classification`, ticketData)
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import { taskService, mlService } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [stats, setStats] = useState({});
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);

  useEffect(() => {
    loadDashboard();
  }, []);

  const loadDashboard = async () => {
    try {
      const response = await taskService.getTasks({ assignedTo: 'me' });
      setTasks(response.data);
      
      calculateStats(response.data);
      analyzeBurnout(response.data);
    } catch (error) {
      console.error('Error loading dashboard:', error);
    }
  };

  const calculateStats = (taskList) => {
    const total = taskList.length;
    const completed = taskList.filter(t => t.status === 'done').length;
    const inProgress = taskList.filter(t => t.status === 'in-progress').length;
    const todo = taskList.filter(t => t.status === 'todo').length;
    
    setStats({
      total,
      completed,
      inProgress,
      todo,
      completionRate: total > 0 ? (completed / total) : 0
    });
  };

  const analyzeBurnout = async (taskList) => {
    try {
      const completedTasks = taskList.filter(t => t.status === 'done');
      const avgTime = completedTasks.reduce((sum, t) => sum + (t.timeSpent || 0), 0) / 
                      (completedTasks.length || 1);
      
      const lateSubmissions = taskList.filter(t => 
        t.dueDate && new Date(t.completedAt) > new Date(t.dueDate)
      ).length;
      
      const behaviorData = {
        user_id: localStorage.getItem('userId'),
        login_frequency: 0.8,
        task_completion_rate: stats.completionRate || 0,
        avg_task_time: avgTime / 3600, // Convert to hours
        failed_login_attempts: 0,
        late_submissions: lateSubmissions,
        active_hours: [8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
      };
      
      const response = await mlService.analyzeBurnout(behaviorData);
      setBurnoutAnalysis(response.data);
    } catch (error) {
      console.error('Error analyzing burnout:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadDashboard();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>User Dashboard</h1>
      
      {/* Stats Cards */}
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-value">{stats.total}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p className="stat-value">{stats.completed}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p className="stat-value">{stats.inProgress}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p className="stat-value">
            {((stats.completionRate || 0) * 100).toFixed(1)}%
          </p>
        </div>
      </div>

      {/* Burnout Alert */}
      {burnoutAnalysis && burnoutAnalysis.risk_level !== 'normal' && (
        <div className={`alert alert-${burnoutAnalysis.risk_level}`}>
          <h4>⚠️ Burnout Warning</h4>
          <p>Risk Level: {burnoutAnalysis.risk_level}</p>
          <ul>
            {burnoutAnalysis.recommendations.filter(Boolean).map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {/* Kanban Board */}
      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.toUpperCase().replace('-', ' ')}</h3>
            <div className="task-list">
              {tasks.filter(t => t.status === status).map(task => (
                <div key={task._id} className="task-card">
                  <h4>{task.title}</h4>
                  <p>{task.description}</p>
                  <div className="task-actions">
                    {status !== 'done' && (
                      <button onClick={() => moveTask(task._id, 
                        status === 'todo' ? 'in-progress' : 'done'
                      )}>
                        Move →
                      </button>
                    )}
                  </div>
                </div>
              ))}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

## Common Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }
    
    req.user = { id: user._id, role: user.role };
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

exports.authorize = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};
```

### Ticket Auto-Classification

```javascript
// backend/routes/tickets.js - Create ticket with AI classification
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/api/ml/ticket-classification`, {
      title,
      description,
      priority
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority: mlResponse.data.suggested_priority,
      category: mlResponse.data.category,
      assignedTo: mlResponse.data.routing,
      createdBy: req.user.id,
      aiClassification: mlResponse.data
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB status
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in Node.js
# backend/config/database.js
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => {
  console.log('MongoDB connected successfully');
}).catch(err => {
  console.error('MongoDB connection error:', err);
  process.exit(1);
});
```

### JWT Token Expiry

```javascript
// frontend/src/services/api.js - Handle token refresh
api.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token expired
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Not Responding

```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Test ML endpoint directly
curl -X POST http://localhost:8000/api/ml/risk-prediction \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "test123",
    "login_frequency": 0.8,
    "task_completion_rate": 0.9,
    "avg_task_time": 4.5,
    "failed_login_attempts": 0,
    "late_submissions": 2,
    "active_hours": [9, 10, 11, 12, 13, 14, 15, 16, 17]
  }'
```

### CORS Issues

```javascript
// backend/server.js - Configure CORS
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));

// ml-service/main.py - FastAPI CORS
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Performance Optimization

```javascript
// Add caching for frequently accessed data
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

router.get('/api/analytics/dashboard', authenticate, async (req, res) => {
  const cacheKey = `dashboard_${req.user.id}`;
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  const data = await generateDashboardData(req.user.id);
  cache.set(cacheKey, data);
  res.json(data);
});
```

This skill provides comprehensive guidance for setting up and using the Enterprise User Management System with AI Analytics, including backend APIs, ML service integration, frontend components, and common troubleshooting scenarios.
