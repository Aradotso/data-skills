---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI"
  - "integrate AI analytics for user management"
  - "create task management with burnout detection"
  - "implement smart ticket routing system"
  - "build user dashboard with AI insights"
  - "add anomaly detection to user system"
  - "configure AI-powered task analytics"
  - "deploy enterprise management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- User authentication and role-based access control (Admin/User)
- Task management with Kanban boards and time tracking
- AI-powered ticket classification and routing
- Predictive analytics (risk detection, burnout analysis, anomaly detection)
- Real-time dashboards and notifications

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance
- Git

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
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

```env
MODEL_PATH=./models
LOG_LEVEL=info
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

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture

### Backend API Structure

The Node.js backend exposes RESTful APIs:

**Authentication:**
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login and get JWT token

**Users (Admin only):**
- `GET /api/users` - List all users
- `GET /api/users/:id` - Get user details
- `PUT /api/users/:id` - Update user
- `DELETE /api/users/:id` - Delete user

**Tasks:**
- `GET /api/tasks` - Get tasks (filtered by role)
- `POST /api/tasks` - Create task (Admin)
- `PUT /api/tasks/:id` - Update task status
- `DELETE /api/tasks/:id` - Delete task

**Tickets:**
- `GET /api/tickets` - Get support tickets
- `POST /api/tickets` - Create ticket
- `PUT /api/tickets/:id` - Update ticket

**Analytics:**
- `GET /api/analytics/dashboard` - Dashboard metrics
- `GET /api/analytics/user/:id` - User-specific analytics

### ML Service API Structure

FastAPI endpoints for AI features:

- `POST /classify-ticket` - AI ticket classification
- `POST /detect-risk` - User risk prediction
- `POST /detect-anomaly` - Anomaly detection
- `POST /predict-burnout` - Burnout analysis
- `POST /predict-project-delay` - Project delay prediction
- `GET /health` - Service health check

## Code Examples

### Backend: User Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Backend: Task Management Route

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get tasks (users see only their tasks, admins see all)
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { assignedTo: req.user.id };
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (Admin only)
router.post('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task status
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    Object.assign(task, req.body);
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

### Backend: MongoDB Models

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
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
  dueDate: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  tags: [String]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
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
  aiClassification: {
    category: String,
    confidence: Number
  }
}, {
  timestamps: true
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### ML Service: FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise User Management AI Service")

# Models will be loaded or trained on startup
models = {}

class TicketData(BaseModel):
    title: str
    description: str
    user_history: Optional[List[str]] = []

class RiskData(BaseModel):
    failed_logins: int
    unusual_hours: int
    data_access_volume: int
    location_changes: int

class BurnoutData(BaseModel):
    tasks_assigned: int
    hours_worked: float
    tasks_overdue: int
    days_since_break: int

class AnomalyData(BaseModel):
    login_time_hour: int
    data_accessed_gb: float
    failed_attempts: int
    ip_change: bool

@app.on_event("startup")
async def load_models():
    """Load pre-trained models or initialize new ones"""
    model_path = os.getenv("MODEL_PATH", "./models")
    
    # Initialize with dummy models if not found
    if not os.path.exists(f"{model_path}/ticket_classifier.pkl"):
        models['ticket_classifier'] = RandomForestClassifier(n_estimators=100)
    else:
        models['ticket_classifier'] = joblib.load(f"{model_path}/ticket_classifier.pkl")

@app.post("/classify-ticket")
async def classify_ticket(data: TicketData):
    """
    Classify support ticket into categories using AI
    """
    try:
        # Simple rule-based classification (can be replaced with ML model)
        text = f"{data.title} {data.description}".lower()
        
        category = "general"
        confidence = 0.5
        
        if any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            confidence = 0.85
        elif any(word in text for word in ["urgent", "critical", "emergency"]):
            category = "urgent"
            confidence = 0.90
        elif any(word in text for word in ["payment", "invoice", "billing", "charge"]):
            category = "billing"
            confidence = 0.80
        
        # Determine priority based on classification
        priority = "high" if category == "urgent" else "medium"
        
        return {
            "category": category,
            "confidence": confidence,
            "priority": priority,
            "suggested_action": f"Route to {category} team"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-risk")
async def detect_risk(data: RiskData):
    """
    Predict user risk level based on behavior patterns
    """
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        risk_score += min(data.failed_logins * 10, 30)
        risk_score += min(data.unusual_hours * 5, 20)
        risk_score += min(data.data_access_volume * 2, 30)
        risk_score += min(data.location_changes * 8, 20)
        
        risk_level = "low"
        if risk_score > 70:
            risk_level = "high"
        elif risk_score > 40:
            risk_level = "medium"
        
        recommendations = []
        if data.failed_logins > 3:
            recommendations.append("Investigate failed login attempts")
        if data.unusual_hours > 5:
            recommendations.append("Review after-hours activity")
        if data.data_access_volume > 50:
            recommendations.append("Check data access patterns")
        
        return {
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": data.failed_logins,
                "unusual_hours": data.unusual_hours,
                "data_access": data.data_access_volume,
                "location_changes": data.location_changes
            },
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(data: BurnoutData):
    """
    Predict employee burnout risk based on workload metrics
    """
    try:
        burnout_score = 0
        
        # Factor in various workload indicators
        if data.tasks_assigned > 20:
            burnout_score += 25
        elif data.tasks_assigned > 15:
            burnout_score += 15
        
        if data.hours_worked > 50:
            burnout_score += 30
        elif data.hours_worked > 40:
            burnout_score += 15
        
        burnout_score += min(data.tasks_overdue * 5, 25)
        burnout_score += min(data.days_since_break * 2, 20)
        
        risk_level = "low"
        if burnout_score > 60:
            risk_level = "high"
        elif burnout_score > 35:
            risk_level = "medium"
        
        suggestions = []
        if data.tasks_assigned > 15:
            suggestions.append("Redistribute tasks to balance workload")
        if data.hours_worked > 45:
            suggestions.append("Consider reducing overtime hours")
        if data.days_since_break > 30:
            suggestions.append("Encourage time off")
        
        return {
            "burnout_score": min(burnout_score, 100),
            "risk_level": risk_level,
            "workload_metrics": {
                "tasks": data.tasks_assigned,
                "hours": data.hours_worked,
                "overdue": data.tasks_overdue,
                "days_no_break": data.days_since_break
            },
            "suggestions": suggestions
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(data: AnomalyData):
    """
    Detect anomalous user behavior for security monitoring
    """
    try:
        anomaly_score = 0
        anomalies_detected = []
        
        # Check for unusual login times (outside 6 AM - 10 PM)
        if data.login_time_hour < 6 or data.login_time_hour > 22:
            anomaly_score += 20
            anomalies_detected.append("Unusual login time")
        
        # Check for excessive data access
        if data.data_accessed_gb > 10:
            anomaly_score += 30
            anomalies_detected.append("High data access volume")
        
        # Failed login attempts
        if data.failed_attempts > 2:
            anomaly_score += 25
            anomalies_detected.append("Multiple failed login attempts")
        
        # IP address change
        if data.ip_change:
            anomaly_score += 15
            anomalies_detected.append("Login from new location")
        
        is_anomalous = anomaly_score > 40
        
        return {
            "is_anomalous": is_anomalous,
            "anomaly_score": anomaly_score,
            "anomalies": anomalies_detected,
            "severity": "high" if anomaly_score > 60 else "medium" if anomaly_score > 30 else "low",
            "action_required": is_anomalous
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise User Management AI"}
```

### Frontend: Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check for existing token
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchCurrentUser = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Frontend: Task Management Component

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`);
      const tasksByStatus = {
        todo: [],
        inprogress: [],
        done: []
      };
      
      response.data.forEach(task => {
        tasksByStatus[task.status].push(task);
      });
      
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
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

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <div className="board-column">
        <h3>To Do</h3>
        <div 
          className="task-list"
          onDrop={(e) => handleDrop(e, 'todo')}
          onDragOver={handleDragOver}
        >
          {tasks.todo.map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onDragStart={handleDragStart}
            />
          ))}
        </div>
      </div>

      <div className="board-column">
        <h3>In Progress</h3>
        <div 
          className="task-list"
          onDrop={(e) => handleDrop(e, 'inprogress')}
          onDragOver={handleDragOver}
        >
          {tasks.inprogress.map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onDragStart={handleDragStart}
            />
          ))}
        </div>
      </div>

      <div className="board-column">
        <h3>Done</h3>
        <div 
          className="task-list"
          onDrop={(e) => handleDrop(e, 'done')}
          onDragOver={handleDragOver}
        >
          {tasks.done.map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onDragStart={handleDragStart}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onDragStart }) => {
  return (
    <div 
      className={`task-card priority-${task.priority}`}
      draggable
      onDragStart={(e) => onDragStart(e, task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className="priority">{task.priority}</span>
        {task.dueDate && (
          <span className="due-date">
            Due: {new Date(task.dueDate).toLocaleDateString()}
          </span>
        )}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### Frontend: AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    burnout: null,
    risk: null,
    anomaly: null
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data from backend
      const userResponse = await axios.get(`${API_URL}/api/analytics/user/${userId}`);
      const userData = userResponse.data;

      // Get AI predictions from ML service
      const [burnoutRes, riskRes, anomalyRes] = await Promise.all([
        axios.post(`${ML_API_URL}/predict-burnout`, {
          tasks_assigned: userData.tasksAssigned,
          hours_worked: userData.hoursWorked,
          tasks_overdue: userData.tasksOverdue,
          days_since_break: userData.daysSinceBreak
        }),
        axios.post(`${ML_API_URL}/detect-risk`, {
          failed_logins: userData.failedLogins,
          unusual_hours: userData.unusualHours,
          data_access_volume: userData.dataAccessVolume,
          location_changes: userData.locationChanges
        }),
        axios.post(`${ML_API_URL}/detect-anomaly`, {
          login_time_hour: new Date().getHours(),
          data_accessed_gb: userData.dataAccessedToday,
          failed_attempts: userData.failedAttemptsToday,
          ip_change: userData.ipChanged
        })
      ]);

      setAnalytics({
        burnout: burnoutRes.data,
        risk: riskRes.data,
        anomaly: anomalyRes.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>

      <div className="analytics-grid">
        {/* Burnout Analysis */}
        <div className={`analytics-card burnout-${analytics.burnout?.risk_level}`}>
          <h3>Burnout Risk</h3>
          <div className="score">{analytics.burnout?.burnout_score}/100</div>
          <div className="level">{analytics.burnout?.risk_level}</div>
          <ul className="suggestions">
            {analytics.burnout?.suggestions.map((suggestion, i) => (
              <li key={i}>{suggestion}</li>
            ))}
          </ul>
        </div>

        {/* Risk Detection */}
        <div className={`analytics-card risk-${analytics.risk?.risk_level}`}>
          <h3>Security Risk</h3>
          <div className="score">{analytics.risk?.risk_score}/100</div>
          <div className="level">{analytics.risk?.risk_level}</div>
          <ul className="recommendations">
            {analytics.risk?.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>

        {/* Anomaly Detection */}
        <div className={`analytics-card anomaly-${analytics.anomaly?.severity}`}>
          <h3>Anomaly Detection</h3>
          <div className="status">
            {analytics.anomaly?.is_anomalous ? '⚠️ Anomaly Detected' : '✓ Normal'}
          </div>
          <ul className="anomalies">
            {analytics.anomaly?.anomalies.map((anomaly, i) => (
              <li key={i}>{anomaly}</li>
            ))}
          </ul>
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

### Frontend: Ticket Creation with AI Classification

```javascript
// frontend/src/components/CreateTicket.js
import React, { useState } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const CreateTicket = ({ onTicketCreated }) => {
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const classifyTicket = async () => {
    if (!formData.title || !formData.description) return;

    try {
      setLoading(true);
      const response = await axios.post(`${ML_API_URL}/classify-ticket`, {
        title: formData.title,
        description: formData.description
      });
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('Error classifying ticket:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      const ticketData = {
        ...formData,
        category: aiSuggestion?.category || 'general',
        priority: aiSuggestion?.priority || 'medium',
        aiClassification: aiSuggestion ? {
          category: aiSuggestion.category,
          confidence: aiSuggestion.confidence
        } : null
      };

      const response = await axios.post(`${API_URL}/api/tickets`, ticketData);
      onTicketCreated(response.data);
      
      // Reset form
      setFormData({ title: '', description: '' });
      setAiSuggestion(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleChange}
            onBlur={classifyTicket}
            required
          />
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleChange}
            onBlur={classifyTicket}
            rows="4"
            required
          />
        </div>

        {aiSuggestion && (
          <div className="ai-suggestion">
            <h4>🤖 AI Classification</h4>
            <p><strong>Category:</strong> {aiSuggestion.category}</p>
            <p><strong>Priority:</strong> {aiSuggestion.priority}</p>
            <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(0)}%</p>
            <p><strong>Suggested Action
