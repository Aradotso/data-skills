---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system with AI analytics
  - implement AI-powered user management and task tracking
  - integrate AI ticket classification and risk prediction
  - build a user management dashboard with anomaly detection
  - set up JWT authentication with role-based access control
  - create AI-driven burnout detection and project insights
  - configure the ML service for predictive analytics
  - deploy enterprise user management with AI features
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you work with an enterprise-grade user management system that combines traditional CRUD operations with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive project insights. The system uses React for frontend, Node.js for backend, FastAPI for ML services, and MongoDB for data persistence.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Secure authentication, role-based access control, and user lifecycle management
- **Task Management**: Kanban boards, time tracking, and progress monitoring
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Centralized monitoring, audit logs, and organizational insights

## Installation

### Prerequisites
- Node.js (v14+)
- Python (3.8+)
- MongoDB (running instance)

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

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

## Key Architecture Components

### Backend API Structure (Node.js)

**User Authentication and Management:**

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
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Create user
    const user = new User({
      username,
      email,
      password, // Should be hashed in User model pre-save hook
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Middleware for JWT Authentication:**

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token, authorization denied' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

**Task Management API:**

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      status: 'todo',
      assignedTo: assignedTo || req.user.userId,
      createdBy: req.user.userId,
      dueDate
    });
    
    await task.save();
    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
    
    task.status = status;
    task.timeTracking = task.timeTracking || {};
    
    if (status === 'in-progress' && !task.timeTracking.startTime) {
      task.timeTracking.startTime = new Date();
    } else if (status === 'done' && !task.timeTracking.endTime) {
      task.timeTracking.endTime = new Date();
    }
    
    await task.save();
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### ML Service API (FastAPI)

**AI Ticket Classification:**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# Models storage
models = {}

class TicketRequest(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class RiskAnalysisRequest(BaseModel):
    userId: str
    failedLogins: int
    tasksCompleted: int
    tasksOverdue: int
    avgTaskCompletionTime: float

@app.on_event("startup")
async def load_models():
    """Load pre-trained models on startup"""
    model_path = os.getenv("MODEL_PATH", "./models")
    
    # Load ticket classifier if exists
    try:
        models['ticket_vectorizer'] = joblib.load(f"{model_path}/ticket_vectorizer.pkl")
        models['ticket_classifier'] = joblib.load(f"{model_path}/ticket_classifier.pkl")
    except:
        # Initialize default models
        models['ticket_vectorizer'] = TfidfVectorizer(max_features=1000)
        models['ticket_classifier'] = MultinomialNB()

@app.post("/api/ai/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """Classify support ticket category and priority"""
    try:
        text = f"{ticket.title} {ticket.description}"
        
        # Simple rule-based classification for demo
        categories = {
            'bug': ['error', 'crash', 'broken', 'not working', 'issue', 'problem'],
            'feature': ['add', 'new', 'feature', 'enhancement', 'improve'],
            'access': ['login', 'password', 'access', 'permission', 'locked'],
            'performance': ['slow', 'lag', 'performance', 'speed', 'timeout']
        }
        
        text_lower = text.lower()
        category = 'general'
        confidence = 0.5
        
        for cat, keywords in categories.items():
            if any(keyword in text_lower for keyword in keywords):
                category = cat
                confidence = 0.85
                break
        
        # Priority suggestion
        priority_keywords = {
            'high': ['urgent', 'critical', 'asap', 'immediately', 'down', 'crash'],
            'low': ['minor', 'cosmetic', 'suggestion', 'when possible']
        }
        
        suggested_priority = ticket.priority
        for prio, keywords in priority_keywords.items():
            if any(keyword in text_lower for keyword in keywords):
                suggested_priority = prio
                break
        
        return {
            "category": category,
            "confidence": confidence,
            "suggestedPriority": suggested_priority,
            "estimatedResolutionTime": _estimate_resolution_time(category, suggested_priority)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def _estimate_resolution_time(category: str, priority: str) -> str:
    """Estimate ticket resolution time"""
    base_times = {
        'bug': 24,
        'feature': 72,
        'access': 4,
        'performance': 48,
        'general': 24
    }
    
    multipliers = {
        'high': 0.5,
        'medium': 1.0,
        'low': 2.0
    }
    
    hours = base_times.get(category, 24) * multipliers.get(priority, 1.0)
    return f"{int(hours)} hours"

@app.post("/api/ai/risk-analysis")
async def analyze_risk(request: RiskAnalysisRequest):
    """Analyze user risk based on behavior patterns"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        factors = []
        
        # Failed logins
        if request.failedLogins > 5:
            risk_score += 30
            factors.append("Multiple failed login attempts")
        elif request.failedLogins > 2:
            risk_score += 15
        
        # Overdue tasks
        overdue_ratio = request.tasksOverdue / max(request.tasksCompleted, 1)
        if overdue_ratio > 0.5:
            risk_score += 25
            factors.append("High percentage of overdue tasks")
        elif overdue_ratio > 0.2:
            risk_score += 10
        
        # Task completion time
        if request.avgTaskCompletionTime > 168:  # > 1 week
            risk_score += 20
            factors.append("Slow task completion times")
        
        # Determine risk level
        if risk_score >= 60:
            level = "high"
            recommendation = "Immediate review required"
        elif risk_score >= 30:
            level = "medium"
            recommendation = "Monitor closely"
        else:
            level = "low"
            recommendation = "Normal activity"
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": level,
            "factors": factors,
            "recommendation": recommendation
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/burnout-detection")
async def detect_burnout(data: dict):
    """Detect employee burnout based on workload patterns"""
    try:
        hours_worked = data.get('hoursWorked', 0)
        tasks_assigned = data.get('tasksAssigned', 0)
        tasks_completed = data.get('tasksCompleted', 0)
        days_since_break = data.get('daysSinceBreak', 0)
        
        burnout_score = 0
        indicators = []
        
        # Excessive hours
        if hours_worked > 50:
            burnout_score += 30
            indicators.append("Working excessive hours")
        
        # High task load
        completion_rate = tasks_completed / max(tasks_assigned, 1)
        if completion_rate < 0.5 and tasks_assigned > 10:
            burnout_score += 25
            indicators.append("Low task completion rate with high workload")
        
        # No breaks
        if days_since_break > 30:
            burnout_score += 25
            indicators.append("Extended period without time off")
        
        # Determine burnout risk
        if burnout_score >= 60:
            risk = "high"
            action = "Urgent intervention needed - schedule meeting with manager"
        elif burnout_score >= 30:
            risk = "medium"
            action = "Consider workload redistribution"
        else:
            risk = "low"
            action = "Continue monitoring"
        
        return {
            "burnoutScore": min(burnout_score, 100),
            "riskLevel": risk,
            "indicators": indicators,
            "recommendedAction": action
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/predict-delay")
async def predict_project_delay(data: dict):
    """Predict project delay probability"""
    try:
        tasks_total = data.get('tasksTotal', 0)
        tasks_completed = data.get('tasksCompleted', 0)
        days_elapsed = data.get('daysElapsed', 0)
        days_remaining = data.get('daysRemaining', 0)
        
        # Calculate progress rate
        completion_rate = tasks_completed / max(tasks_total, 1)
        expected_completion = days_elapsed / max(days_elapsed + days_remaining, 1)
        
        delay_probability = 0
        factors = []
        
        if completion_rate < expected_completion - 0.2:
            delay_probability += 60
            factors.append("Behind schedule")
        elif completion_rate < expected_completion:
            delay_probability += 30
        
        # Calculate estimated delay
        if completion_rate > 0:
            total_days_needed = days_elapsed / completion_rate
            estimated_delay = max(0, total_days_needed - (days_elapsed + days_remaining))
        else:
            estimated_delay = days_remaining
        
        return {
            "delayProbability": min(delay_probability, 100),
            "estimatedDelayDays": int(estimated_delay),
            "completionRate": round(completion_rate * 100, 2),
            "factors": factors,
            "recommendation": "Increase resources" if delay_probability > 50 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "AI Analytics"}
```

### Frontend Integration (React)

**API Service Layer:**

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instances
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

const mlApi = axios.create({
  baseURL: ML_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth services
export const authService = {
  login: async (email, password) => {
    const response = await api.post('/api/auth/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    return response.data;
  },
  
  register: async (userData) => {
    const response = await api.post('/api/auth/register', userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

// Task services
export const taskService = {
  getTasks: async () => {
    const response = await api.get('/api/tasks');
    return response.data.tasks;
  },
  
  createTask: async (taskData) => {
    const response = await api.post('/api/tasks', taskData);
    return response.data.task;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await api.patch(`/api/tasks/${taskId}/status`, { status });
    return response.data.task;
  }
};

// AI services
export const aiService = {
  classifyTicket: async (ticketData) => {
    const response = await mlApi.post('/api/ai/classify-ticket', ticketData);
    return response.data;
  },
  
  analyzeRisk: async (userData) => {
    const response = await mlApi.post('/api/ai/risk-analysis', userData);
    return response.data;
  },
  
  detectBurnout: async (workloadData) => {
    const response = await mlApi.post('/api/ai/burnout-detection', workloadData);
    return response.data;
  },
  
  predictDelay: async (projectData) => {
    const response = await mlApi.post('/api/ai/predict-delay', projectData);
    return response.data;
  }
};

export default api;
```

**Kanban Board Component:**

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const data = await taskService.getTasks();
      setTasks(data);
    } catch (error) {
      console.error('Error loading tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const columns = ['todo', 'in-progress', 'done'];
  const columnTitles = {
    'todo': 'To Do',
    'in-progress': 'In Progress',
    'done': 'Done'
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {columns.map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{columnTitles[status]}</h3>
          <div className="task-list">
            {tasks
              .filter(task => task.status === status)
              .map(task => (
                <div
                  key={task._id}
                  className={`task-card priority-${task.priority}`}
                  draggable
                  onDragStart={(e) => handleDragStart(e, task._id)}
                >
                  <h4>{task.title}</h4>
                  <p>{task.description}</p>
                  <span className="priority-badge">{task.priority}</span>
                  {task.dueDate && (
                    <span className="due-date">
                      Due: {new Date(task.dueDate).toLocaleDateString()}
                    </span>
                  )}
                </div>
              ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

**AI Analytics Dashboard:**

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { aiService } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    loading: true
  });

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      // Fetch user metrics (would come from backend in real app)
      const userMetrics = {
        userId,
        failedLogins: 1,
        tasksCompleted: 15,
        tasksOverdue: 3,
        avgTaskCompletionTime: 48,
        hoursWorked: 45,
        tasksAssigned: 20,
        daysSinceBreak: 14
      };

      const [risk, burnout] = await Promise.all([
        aiService.analyzeRisk(userMetrics),
        aiService.detectBurnout(userMetrics)
      ]);

      setAnalytics({ risk, burnout, loading: false });
    } catch (error) {
      console.error('Error loading analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  if (analytics.loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      {analytics.risk && (
        <div className={`analytics-card risk-${analytics.risk.riskLevel}`}>
          <h3>Risk Analysis</h3>
          <div className="score">
            Risk Score: {analytics.risk.riskScore}/100
          </div>
          <div className="level">Level: {analytics.risk.riskLevel.toUpperCase()}</div>
          <ul className="factors">
            {analytics.risk.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
          <p className="recommendation">{analytics.risk.recommendation}</p>
        </div>
      )}

      {analytics.burnout && (
        <div className={`analytics-card burnout-${analytics.burnout.riskLevel}`}>
          <h3>Burnout Detection</h3>
          <div className="score">
            Burnout Score: {analytics.burnout.burnoutScore}/100
          </div>
          <div className="level">Risk: {analytics.burnout.riskLevel.toUpperCase()}</div>
          <ul className="indicators">
            {analytics.burnout.indicators.map((indicator, idx) => (
              <li key={idx}>{indicator}</li>
            ))}
          </ul>
          <p className="action">{analytics.burnout.recommendedAction}</p>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

**User Model:**

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
    select: false
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
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Task Model:**

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
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
  dueDate: Date,
  timeTracking: {
    startTime: Date,
    endTime: Date,
    totalHours: Number
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

## Common Patterns and Use Cases

### Admin User Management

```javascript
// backend/routes/admin.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get all users (admin only)
router.get('/users', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, users });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update user role
router.patch('/users/:id/role', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { role } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { role },
      { new: true }
    ).select('-password');
    
    res.json({ success: true, user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Delete user
router.delete('/users/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    await User.findById
