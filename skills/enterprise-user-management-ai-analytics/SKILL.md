---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management using React, Node.js, and FastAPI ML services
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with kanban board"
  - "add risk detection and anomaly detection"
  - "create admin dashboard with AI insights"
  - "build user management with JWT authentication"
  - "configure ML service for burnout prediction"
  - "implement ticket classification system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket creation, classification, and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide insights and monitoring

The system consists of three main components:
- **Frontend**: React.js application
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI service with scikit-learn and River for real-time ML

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
# Edit .env with your MongoDB URI and JWT secret

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
cp .env.example .env
# Configure API endpoints
```

### Environment Configuration

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**:
```bash
MODEL_PATH=./models
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
API_KEY=your_ml_service_api_key
```

## Running the System

### Start All Services

**Terminal 1 - Backend**:
```bash
cd backend
npm start
# Runs on http://localhost:5000
```

**Terminal 2 - ML Service**:
```bash
cd ml-service
source venv/bin/activate
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

**Terminal 3 - Frontend**:
```bash
cd frontend
npm start
# Runs on http://localhost:3000
```

## Backend API Patterns

### Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

module.exports = authMiddleware;
```

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const authMiddleware = require('../middleware/auth');
const adminMiddleware = require('../middleware/admin');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { email, password, firstName, lastName, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    const user = new User({
      email,
      password, // Should be hashed in User model pre-save hook
      firstName,
      lastName,
      role: role || 'user'
    });
    
    await user.save();
    const token = user.generateAuthToken();
    
    res.status(201).json({ user, token });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get all users (Admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.patch('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const updates = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const authMiddleware = require('../middleware/auth');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.userId
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.userId })
      .populate('createdBy', 'firstName lastName')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: new Date() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const authMiddleware = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category: mlResponse.data.category,
      predictedPriority: mlResponse.data.priority,
      createdBy: req.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.userRole === 'admin' 
      ? {} 
      : { createdBy: req.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'firstName lastName email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models at startup
@app.on_event("startup")
async def load_models():
    global ticket_classifier, risk_detector, burnout_predictor
    try:
        ticket_classifier = joblib.load("./models/ticket_classifier.pkl")
        risk_detector = joblib.load("./models/risk_detector.pkl")
        burnout_predictor = joblib.load("./models/burnout_predictor.pkl")
    except FileNotFoundError:
        print("Warning: Models not found. Initialize with default models.")
        ticket_classifier = None
        risk_detector = None
        burnout_predictor = None

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    """Classify support ticket using NLP"""
    try:
        # Feature extraction
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification (replace with trained model)
        categories = {
            "technical": ["bug", "error", "crash", "issue", "problem"],
            "account": ["login", "password", "access", "permission"],
            "feature": ["request", "need", "want", "add"],
            "billing": ["payment", "invoice", "charge", "subscription"]
        }
        
        category_scores = {}
        for cat, keywords in categories.items():
            score = sum(1 for kw in keywords if kw in text)
            category_scores[cat] = score
        
        category = max(category_scores, key=category_scores.get)
        
        # Priority detection
        urgent_words = ["urgent", "critical", "asap", "emergency", "blocked"]
        priority = "high" if any(word in text for word in urgent_words) else "medium"
        
        confidence = category_scores[category] / 10.0
        
        return TicketClassification(
            category=category,
            priority=priority,
            confidence=min(confidence, 1.0)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class UserActivity(BaseModel):
    userId: str
    loginFrequency: int
    failedLogins: int
    dataAccessCount: int
    unusualHours: int
    locationChanges: int

class RiskScore(BaseModel):
    riskLevel: str
    score: float
    factors: List[str]

@app.post("/detect-risk", response_model=RiskScore)
async def detect_risk(activity: UserActivity):
    """Detect user risk based on behavior patterns"""
    try:
        # Calculate risk factors
        risk_score = 0.0
        factors = []
        
        if activity.failedLogins > 3:
            risk_score += 0.3
            factors.append("Multiple failed login attempts")
        
        if activity.unusualHours > 5:
            risk_score += 0.2
            factors.append("Activity during unusual hours")
        
        if activity.locationChanges > 3:
            risk_score += 0.2
            factors.append("Multiple location changes")
        
        if activity.dataAccessCount > 100:
            risk_score += 0.15
            factors.append("High data access volume")
        
        if activity.loginFrequency < 2:
            risk_score += 0.15
            factors.append("Irregular login pattern")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "critical"
        elif risk_score >= 0.4:
            risk_level = "high"
        elif risk_score >= 0.2:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskScore(
            riskLevel=risk_level,
            score=risk_score,
            factors=factors if factors else ["No risk factors detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class WorkloadData(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHours: float
    overtimeHours: int
    missedDeadlines: int

class BurnoutPrediction(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: List[str]

@app.post("/predict-burnout", response_model=BurnoutPrediction)
async def predict_burnout(workload: WorkloadData):
    """Predict employee burnout risk"""
    try:
        score = 0.0
        recommendations = []
        
        # Calculate burnout indicators
        task_completion_rate = workload.tasksCompleted / max(workload.tasksAssigned, 1)
        
        if workload.avgWorkHours > 50:
            score += 0.3
            recommendations.append("Reduce work hours to healthy levels")
        
        if workload.overtimeHours > 20:
            score += 0.25
            recommendations.append("Limit overtime hours")
        
        if workload.missedDeadlines > 3:
            score += 0.2
            recommendations.append("Review task priorities and deadlines")
        
        if task_completion_rate < 0.6:
            score += 0.15
            recommendations.append("Reassess workload distribution")
        
        if workload.tasksAssigned > 15:
            score += 0.1
            recommendations.append("Reduce number of concurrent tasks")
        
        # Determine risk level
        if score >= 0.7:
            burnout_risk = "critical"
        elif score >= 0.5:
            burnout_risk = "high"
        elif score >= 0.3:
            burnout_risk = "medium"
        else:
            burnout_risk = "low"
            recommendations.append("Workload appears balanced")
        
        return BurnoutPrediction(
            burnoutRisk=burnout_risk,
            score=score,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class ProjectData(BaseModel):
    projectId: str
    plannedDuration: int
    currentProgress: float
    teamSize: int
    blockers: int
    completedMilestones: int
    totalMilestones: int

class ProjectInsight(BaseModel):
    delayProbability: float
    estimatedCompletion: str
    recommendations: List[str]

@app.post("/predict-project-delay", response_model=ProjectInsight)
async def predict_project_delay(project: ProjectData):
    """Predict project delay probability"""
    try:
        # Calculate progress metrics
        milestone_completion = project.completedMilestones / max(project.totalMilestones, 1)
        
        delay_factors = 0.0
        recommendations = []
        
        if project.currentProgress < 0.5 and project.plannedDuration < 30:
            delay_factors += 0.3
            recommendations.append("Project is behind schedule - increase resources")
        
        if project.blockers > 3:
            delay_factors += 0.25
            recommendations.append("Address critical blockers immediately")
        
        if project.teamSize < 3:
            delay_factors += 0.15
            recommendations.append("Consider expanding team size")
        
        if milestone_completion < 0.6:
            delay_factors += 0.2
            recommendations.append("Focus on completing pending milestones")
        
        delay_probability = min(delay_factors, 1.0)
        
        # Estimate completion
        days_remaining = project.plannedDuration * (1 + delay_probability * 0.5)
        estimated_date = datetime.now().strftime("%Y-%m-%d")
        
        if not recommendations:
            recommendations.append("Project is on track")
        
        return ProjectInsight(
            delayProbability=delay_probability,
            estimatedCompletion=estimated_date,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL;
const ML_API_BASE_URL = process.env.REACT_APP_ML_API_URL;

// Create axios instance with auth interceptor
const api = axios.create({
  baseURL: API_BASE_URL,
});

api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

export const authService = {
  login: async (email, password) => {
    const response = await api.post('/auth/login', { email, password });
    localStorage.setItem('authToken', response.data.token);
    return response.data;
  },
  
  register: async (userData) => {
    const response = await api.post('/users/register', userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('authToken');
  },
};

export const userService = {
  getAllUsers: () => api.get('/users'),
  getUserById: (id) => api.get(`/users/${id}`),
  updateUser: (id, data) => api.patch(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`),
};

export const taskService = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  deleteTask: (id) => api.delete(`/tasks/${id}`),
};

export const ticketService = {
  getAllTickets: () => api.get('/tickets'),
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  updateTicket: (id, data) => api.patch(`/tickets/${id}`, data),
};

export const mlService = {
  classifyTicket: async (title, description) => {
    const response = await axios.post(`${ML_API_BASE_URL}/classify-ticket`, {
      title,
      description,
    });
    return response.data;
  },
  
  detectRisk: async (activityData) => {
    const response = await axios.post(`${ML_API_BASE_URL}/detect-risk`, activityData);
    return response.data;
  },
  
  predictBurnout: async (workloadData) => {
    const response = await axios.post(`${ML_API_BASE_URL}/predict-burnout`, workloadData);
    return response.data;
  },
  
  predictProjectDelay: async (projectData) => {
    const response = await axios.post(`${ML_API_BASE_URL}/predict-project-delay`, projectData);
    return response.data;
  },
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskService.getMyTasks();
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done'),
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title) => (
    <div
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({tasks[status].length})</h3>
      <div className="task-list">
        {tasks[status].map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => handleDragStart(e, task._id, status)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('inProgress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userService, mlService } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const usersResponse = await userService.getAllUsers();
      setUsers(usersResponse.data);

      // Analyze risk for each user
      const riskPromises = usersResponse.data.map(async (user) => {
        const activityData = {
          userId: user._id,
          loginFrequency: user.loginCount || 0,
          failedLogins: user.failedLogins || 0,
          dataAccessCount: user.dataAccess || 0,
          unusualHours: user.unusualHours || 0,
          locationChanges: user.locationChanges || 0,
        };

        const risk = await mlService.detectRisk(activityData);
        return { userId: user._id, userName: `${user.firstName} ${user.lastName}`, ...risk };
      });

      const risks = await Promise.all(riskPromises);
      setRiskAnalysis(risks);
    } catch (error) {
      console.error('Failed to load dashboard data:', error);
    } finally {
      setLoading(false);
    }
  };

  const getRiskColor = (level) => {
    const colors = {
      critical: '#dc3545',
      high: '#fd7e14',
      medium: '#ffc107',
      low: '#28a745',
    };
    return colors[level] || '#6c757d';
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      <div className="stats-overview">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="stat-number">
            {riskAnalysis.filter(r => r.riskLevel === 'high' || r.riskLevel === 'critical').length}
          </p>
        </div>
      </div>

      <div className="risk-analysis-section">
        <h2>User Risk Analysis</h2>
        <table className="risk-table">
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Level</th>
              <th>Risk Score</th>
              <th>Factors</th>
            </tr>
          </thead>
          <tbody>
            {riskAnalysis.map(risk => (
              <tr key={risk.userId}>
                <td>{risk.userName}</td>
                <td>
                  <span
                    className="risk-badge"
                    style={{ backgroundColor: getRiskColor(risk.riskLevel) }}
                  >
                    {risk.riskLevel.toUpperCase()}
                  </span>
                </td>
                <td>{(risk.score * 100).toFixed(0)}%</td>
                <td>
                  <ul className="factors-list">
                    {risk.factors.map((factor, idx) => (
                      <li key={idx}>{factor}</li>
                    ))}
                  </ul>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
  },
  password: {
    type: String,
    required: true,
    minlength: 6,
  },
  firstName: {
    type: String,
    required: true,
    trim: true,
  },
  lastName: {
    type: String,
    required: true,
    trim: true,
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user',
  },
  isActive: {
    type: Boolean,
    default: true,
  },
  loginCount: {
    type: Number,
    default: 0,
  },
  failedLogins: {
    type: Number,
    default: 0,
  },
  lastLogin: Date,
}, {
  timestamps: true,
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Generate JWT token
userSchema.methods.generateAuthToken = function() {
  return jwt.sign(
    { userId: this._id, role: this.role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRE }
  );
};

// Compare password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
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
    trim: true,
  },
  description: {
    type: String,
    required: true,
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo',
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
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
