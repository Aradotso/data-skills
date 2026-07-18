---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task automation
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered user management system"
  - "create admin dashboard with AI insights"
  - "build task management with burnout detection"
  - "add AI ticket classification and routing"
  - "integrate ML-based anomaly detection for users"
  - "deploy user management system with predictive analytics"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban board, time tracking, and assignment workflows
- **Support Ticketing**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Centralized monitoring, audit logs, and organizational insights

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js/Express) - REST APIs and business logic
3. **ML Service** (FastAPI) - AI/ML models for predictions and analytics

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB
- Git

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup (separate terminal)
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env  # Configure ML service
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (separate terminal)
cd ../frontend
npm install
cp .env.example .env  # Configure API endpoints
npm start  # Runs on http://localhost:3000
```

## Environment Configuration

### Backend (.env)

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
DB_NAME=enterprise_user_mgmt

# JWT Authentication
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRES_IN=24h

# Server
PORT=5000
NODE_ENV=development

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}
```

### ML Service (.env)

```bash
# API Configuration
ML_API_PORT=8000
ML_API_HOST=0.0.0.0

# Model Paths
MODEL_STORAGE_PATH=./models
RISK_MODEL_PATH=./models/risk_model.pkl
ANOMALY_MODEL_PATH=./models/anomaly_model.pkl
BURNOUT_MODEL_PATH=./models/burnout_model.pkl

# Backend API
BACKEND_URL=http://localhost:5000
```

### Frontend (.env)

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Key Backend API Endpoints

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// POST /api/auth/register
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
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// POST /api/auth/login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// GET /api/users - Get all users (Admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// GET /api/users/:id - Get user by ID
router.get('/:id', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// PUT /api/users/:id - Update user
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const { username, email, role } = req.body;
    
    // Only admin can change roles
    if (role && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Unauthorized to change role' });
    }
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// DELETE /api/users/:id - Delete user (Admin only)
router.delete('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
const { authMiddleware } = require('../middleware/auth');

// GET /api/tasks - Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// POST /api/tasks - Create new task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// PUT /api/tasks/:id/status - Update task status
router.put('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: new Date() },
      { new: true }
    ).populate('assignedTo assignedBy', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// POST /api/tasks/:id/time - Track time on task
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### AI Analytics Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional
import pandas as pd
from river import anomaly, compose, preprocessing

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
try:
    risk_model = joblib.load('models/risk_model.pkl')
except:
    risk_model = None

# Online learning anomaly detector
anomaly_detector = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
)

class UserBehavior(BaseModel):
    user_id: str
    login_count: int
    failed_login_attempts: int
    tasks_completed: int
    tasks_overdue: int
    avg_task_duration: float
    last_login_hour: int

class TaskData(BaseModel):
    task_id: str
    assigned_user_id: str
    priority: str
    estimated_hours: float
    actual_hours: Optional[float] = None
    dependencies_count: int
    user_workload: int

class TicketData(BaseModel):
    ticket_id: str
    title: str
    description: str
    priority: str
    user_id: str

# Risk Prediction
@app.post("/api/ml/predict-risk")
async def predict_risk(behavior: UserBehavior):
    """Predict user risk level based on behavior patterns"""
    try:
        features = np.array([[
            behavior.login_count,
            behavior.failed_login_attempts,
            behavior.tasks_completed,
            behavior.tasks_overdue,
            behavior.avg_task_duration,
            behavior.last_login_hour
        ]])
        
        if risk_model:
            risk_score = risk_model.predict_proba(features)[0][1]
        else:
            # Fallback heuristic
            risk_score = (
                behavior.failed_login_attempts * 0.3 +
                behavior.tasks_overdue * 0.2 +
                (1 - min(behavior.tasks_completed / 10, 1)) * 0.3 +
                (abs(behavior.last_login_hour - 12) / 12) * 0.2
            ) / 100
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "user_id": behavior.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": behavior.failed_login_attempts,
                "overdue_tasks": behavior.tasks_overdue,
                "unusual_login_time": abs(behavior.last_login_hour - 12) > 8
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        features = {
            'login_count': behavior.login_count,
            'failed_attempts': behavior.failed_login_attempts,
            'tasks_completed': behavior.tasks_completed,
            'avg_duration': behavior.avg_task_duration
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.6
        
        return {
            "user_id": behavior.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "severity": "high" if score > 0.8 else "medium" if score > 0.6 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/api/ml/detect-burnout")
async def detect_burnout(user_data: dict):
    """Detect employee burnout based on workload and patterns"""
    try:
        tasks_assigned = user_data.get('tasks_assigned', 0)
        tasks_completed = user_data.get('tasks_completed', 0)
        avg_work_hours = user_data.get('avg_work_hours', 8)
        overtime_days = user_data.get('overtime_days', 0)
        task_completion_rate = user_data.get('task_completion_rate', 1.0)
        
        # Calculate burnout score
        burnout_score = (
            (tasks_assigned / max(tasks_completed, 1) - 1) * 0.3 +
            (max(avg_work_hours - 8, 0) / 8) * 0.3 +
            (overtime_days / 30) * 0.2 +
            (1 - task_completion_rate) * 0.2
        )
        
        burnout_risk = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        recommendations = []
        if avg_work_hours > 10:
            recommendations.append("Reduce daily work hours")
        if task_completion_rate < 0.7:
            recommendations.append("Redistribute workload")
        if overtime_days > 15:
            recommendations.append("Mandatory time off recommended")
        
        return {
            "user_id": user_data.get('user_id'),
            "burnout_score": float(burnout_score),
            "burnout_risk": burnout_risk,
            "recommendations": recommendations,
            "metrics": {
                "workload_ratio": tasks_assigned / max(tasks_completed, 1),
                "avg_work_hours": avg_work_hours,
                "overtime_days": overtime_days
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Ticket Classification
@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and suggest routing"""
    try:
        # Simple keyword-based classification
        text = (ticket.title + " " + ticket.description).lower()
        
        categories = {
            "technical": ["bug", "error", "crash", "issue", "problem", "api", "database"],
            "access": ["login", "password", "permission", "access", "role", "auth"],
            "feature": ["feature", "enhancement", "request", "suggestion", "improve"],
            "documentation": ["documentation", "docs", "guide", "tutorial", "help"]
        }
        
        scores = {}
        for category, keywords in categories.items():
            scores[category] = sum(1 for keyword in keywords if keyword in text)
        
        primary_category = max(scores, key=scores.get)
        
        # Routing logic
        routing = {
            "technical": "Engineering Team",
            "access": "IT Support",
            "feature": "Product Team",
            "documentation": "Documentation Team"
        }
        
        return {
            "ticket_id": ticket.ticket_id,
            "category": primary_category,
            "confidence": scores[primary_category] / max(sum(scores.values()), 1),
            "suggested_team": routing[primary_category],
            "priority_adjustment": "escalate" if ticket.priority == "high" else "normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project Delay Prediction
@app.post("/api/ml/predict-delay")
async def predict_delay(task: TaskData):
    """Predict if a task/project will be delayed"""
    try:
        # Feature engineering
        priority_weight = {"high": 3, "medium": 2, "low": 1}[task.priority]
        
        delay_probability = (
            (task.user_workload / 10) * 0.3 +
            (task.dependencies_count / 5) * 0.2 +
            (priority_weight / 3) * 0.2 +
            (max(task.estimated_hours - 8, 0) / 40) * 0.3
        )
        
        delay_probability = min(delay_probability, 1.0)
        
        will_delay = delay_probability > 0.6
        
        return {
            "task_id": task.task_id,
            "will_delay": will_delay,
            "delay_probability": float(delay_probability),
            "estimated_delay_days": int(delay_probability * 5) if will_delay else 0,
            "factors": {
                "high_workload": task.user_workload > 8,
                "many_dependencies": task.dependencies_count > 3,
                "high_complexity": task.estimated_hours > 16
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration Examples

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    
    return response.data;
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

### Task Management Component

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority priority-${task.priority}`}>
          {task.priority}
        </span>
        {task.dueDate && (
          <span className="due-date">
            Due: {new Date(task.dueDate).toLocaleDateString()}
          </span>
        )}
      </div>
      <div className="task-actions">
        {task.status !== 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'todo')}>← To Do</button>
        )}
        {task.status !== 'in_progress' && (
          <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
            In Progress
          </button>
        )}
        {task.status !== 'done' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>Done →</button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <div className="task-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="task-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="task-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user behavior data
      const behaviorResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/users/${userId}/behavior`
      );
      
      // Get AI predictions
      const [riskData, burnoutData, anomalyData] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, 
          behaviorResponse.data
        ),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-burnout`, {
          user_id: userId,
          ...behaviorResponse.data
        }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-anomaly`,
          behaviorResponse.data
        )
      ]);
      
      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data,
        anomaly: anomalyData.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI analytics...</div>;
  if (!analytics) return <div>No analytics available</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-grid">
        {/* Risk Assessment */}
        <div className={`analytics-card risk-${analytics.risk.risk_level}`}>
          <h3>Risk Level</h3>
          <div className="score">{(analytics.risk.risk_score * 100).toFixed(1)}%</div>
          <div className="level">{analytics.risk.risk_level.toUpperCase()}</div>
          <ul className="factors">
            {analytics.risk.factors.failed_logins > 0 && (
              <li>⚠️ {analytics.risk.factors.failed_logins} failed logins</li>
            )}
            {analytics.risk.factors.overdue_tasks > 0 && (
              <li>⚠️ {analytics.risk.factors.overdue_tasks} overdue tasks</li>
            )}
            {analytics.risk.factors.unusual_login_time && (
              <li>⚠️ Unusual login times detected</li>
            )}
          </ul>
        </div>

        {/* Burnout Detection */}
        <div className={`analytics-card burnout-${analytics.burnout.burnout_risk}`}>
          <h3>Burnout Risk</h3>
          <div className="score">{(analytics.burnout.burnout_score * 100).toFixed(1)}%</div>
          <div className="level">{analytics.burnout.burnout_risk.toUpperCase()}</div>
          {analytics.burnout.recommendations.length > 0 && (
            <div className="recommendations">
              <h4>Recommendations:</h4>
              <ul>
                {analytics.burnout.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
          )}
        </div>

        {/* Anomaly Detection */}
        <div className={`analytics-card anomaly-${analytics.anomaly.severity}`}>
          <h3>Behavior Analysis</h3>
          <div className="status">
            {analytics.anomaly.is_anomaly ? '🚨 Anomaly Detected' : '✓ Normal'}
          </div>
          <div className="score">
            Anomaly Score: {(analytics.anomaly.anomaly_score * 100).toFixed(1)}%
          </div>
          <div className="severity">Severity: {analytics.anomaly.severity}</div>
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

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
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'manager', 'user'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  loginAttempts: {
    type: Number,
    default: 0
  },
  metadata: {
    department: String,
    position: String,
    joinDate: Date
  }
}, {
  timestamps: true
});

module.exports = mongoose.model('User
