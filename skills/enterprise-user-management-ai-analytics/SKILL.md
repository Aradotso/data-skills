---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML services
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics with user management
  - implement JWT authentication for user management
  - create AI-powered ticket classification system
  - build admin dashboard with user analytics
  - set up ML service for burnout detection
  - configure MongoDB for user management app
  - implement Kanban board with task tracking
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning-powered insights. It provides:

- **User Management**: Role-based access control, authentication (JWT), and user CRUD operations
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Support Ticketing**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Organization-wide analytics and audit logs

The system uses React for frontend, Node.js/Express for backend, FastAPI for ML services, and MongoDB for data storage.

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
# Edit .env with your configuration
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
cp .env.example .env
npm start  # Runs on http://localhost:3000
```

## Configuration

### Backend Environment Variables (.env)

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/user_management
DB_NAME=user_management

# Authentication
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRE=7d

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

### ML Service Environment Variables (.env)

```bash
# FastAPI
ML_SERVICE_PORT=8000

# Model Configuration
MODEL_PATH=./models
ANOMALY_THRESHOLD=0.75
BURNOUT_THRESHOLD=0.7
RISK_SCORE_THRESHOLD=0.6

# Backend Connection
BACKEND_URL=http://localhost:5000
```

### Frontend Environment Variables (.env)

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Key API Endpoints

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
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
      { expiresIn: process.env.JWT_EXPIRE }
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
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
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
    res.status(500).json({ error: error.message });
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
const { authenticateToken, isAdmin } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', authenticateToken, isAdmin, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
router.get('/:id', authenticateToken, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authenticateToken, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    
    // Check if user can update (self or admin)
    if (req.user.userId !== req.params.id && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Unauthorized' });
    }
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user (admin only)
router.delete('/:id', authenticateToken, isAdmin, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
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
const { authenticateToken } = require('../middleware/auth');

// Get all tasks for user
router.get('/my-tasks', authenticateToken, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authenticateToken, async (req, res) => {
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
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticateToken, async (req, res) => {
  try {
    const { status } = req.body;
    
    if (!['todo', 'in-progress', 'done'].includes(status)) {
      return res.status(400).json({ error: 'Invalid status' });
    }
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo assignedBy', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time-entry', authenticateToken, async (req, res) => {
  try {
    const { duration, notes } = req.body;
    
    const task = await Task.findById(req.params.id);
    task.timeTracking.push({
      user: req.user.userId,
      duration,
      notes,
      date: Date.now()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { authenticateToken } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Create ticket
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      createdBy: req.user.userId,
      status: 'open'
    });
    
    // AI classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        {
          title,
          description
        }
      );
      
      ticket.category = mlResponse.data.category;
      ticket.suggestedAssignee = mlResponse.data.suggested_assignee;
      ticket.urgencyScore = mlResponse.data.urgency_score;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
      // Continue without AI classification
    }
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authenticateToken, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update ticket
router.patch('/:id', authenticateToken, async (req, res) => {
  try {
    const { status, assignedTo, resolution } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, assignedTo, resolution, updatedAt: Date.now() },
      { new: true }
    ).populate('createdBy assignedTo', 'username email');
    
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from sklearn.feature_extraction.text import TfidfVectorizer
import pickle
import os

app = FastAPI(title="User Management ML Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load or initialize)
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Request models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_activity_count: int
    access_violations: int
    after_hours_access: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    missed_deadlines: int

class AnomalyDetectionRequest(BaseModel):
    login_time: int  # hour of day
    login_location: str
    failed_attempts: int
    session_duration: float

# Ticket Classification
@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-powered ticket classification and routing"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (can be replaced with ML model)
        category = "general"
        urgency_score = 0.5
        
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = "technical"
            urgency_score = 0.8
        elif any(word in text for word in ['password', 'access', 'login', 'permission']):
            category = "access"
            urgency_score = 0.7
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = "feature_request"
            urgency_score = 0.4
        elif any(word in text for word in ['urgent', 'critical', 'emergency']):
            urgency_score = 0.9
        
        # Suggest assignee based on category
        assignee_map = {
            "technical": "tech_support",
            "access": "admin",
            "feature_request": "product_team",
            "general": "support"
        }
        
        return {
            "category": category,
            "urgency_score": urgency_score,
            "suggested_assignee": assignee_map.get(category, "support")
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk Prediction
@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict security risk based on user behavior"""
    try:
        # Calculate risk score
        risk_score = 0.0
        risk_factors = []
        
        if request.failed_logins > 3:
            risk_score += 0.3
            risk_factors.append("Multiple failed login attempts")
        
        if request.unusual_activity_count > 5:
            risk_score += 0.2
            risk_factors.append("Unusual activity detected")
        
        if request.access_violations > 0:
            risk_score += 0.4
            risk_factors.append("Access policy violations")
        
        if request.after_hours_access > 10:
            risk_score += 0.1
            risk_factors.append("Frequent after-hours access")
        
        risk_score = min(risk_score, 1.0)
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": risk_score,
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "recommendation": "Monitor closely" if risk_level == "high" else "Normal monitoring"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutAnalysisRequest):
    """Detect employee burnout based on workload metrics"""
    try:
        burnout_score = 0.0
        indicators = []
        
        # Task completion rate
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        if completion_rate < 0.7:
            burnout_score += 0.2
            indicators.append("Low task completion rate")
        
        # Overtime
        if request.overtime_hours > 20:
            burnout_score += 0.3
            indicators.append("Excessive overtime")
        
        # Missed deadlines
        if request.missed_deadlines > 3:
            burnout_score += 0.3
            indicators.append("Frequent missed deadlines")
        
        # Average task duration
        if request.avg_task_duration > 8:
            burnout_score += 0.2
            indicators.append("Extended task completion times")
        
        burnout_score = min(burnout_score, 1.0)
        
        burnout_level = "low"
        if burnout_score > 0.7:
            burnout_level = "high"
        elif burnout_score > 0.4:
            burnout_level = "medium"
        
        return {
            "user_id": request.user_id,
            "burnout_score": burnout_score,
            "burnout_level": burnout_level,
            "indicators": indicators,
            "recommendation": "Reduce workload and schedule 1-on-1" if burnout_level == "high" 
                            else "Continue monitoring"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        is_anomaly = False
        anomaly_reasons = []
        
        # Unusual login time
        if request.login_time < 6 or request.login_time > 22:
            is_anomaly = True
            anomaly_reasons.append("Login outside normal hours")
        
        # Failed attempts
        if request.failed_attempts > 2:
            is_anomaly = True
            anomaly_reasons.append("Multiple failed login attempts")
        
        # Unusual session duration
        if request.session_duration > 12 or request.session_duration < 0.1:
            is_anomaly = True
            anomaly_reasons.append("Unusual session duration")
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": len(anomaly_reasons) / 3.0,
            "reasons": anomaly_reasons,
            "action": "Block and notify admin" if is_anomaly else "Allow"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Health check
@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

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
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  const register = async (userData) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/register`, userData);
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout, register }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`);
      const categorized = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
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
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
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
    <div className="kanban-board">
      <Column
        title="To Do"
        tasks={tasks.todo}
        status="todo"
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        onDragStart={handleDragStart}
      />
      <Column
        title="In Progress"
        tasks={tasks.inProgress}
        status="in-progress"
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        onDragStart={handleDragStart}
      />
      <Column
        title="Done"
        tasks={tasks.done}
        status="done"
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        onDragStart={handleDragStart}
      />
    </div>
  );
};

const Column = ({ title, tasks, status, onDrop, onDragOver, onDragStart }) => (
  <div
    className="kanban-column"
    onDrop={(e) => onDrop(e, status)}
    onDragOver={onDragOver}
  >
    <h3>{title} ({tasks.length})</h3>
    <div className="task-list">
      {tasks.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => onDragStart(e, task._id)}
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
      ))}
    </div>
  </div>
);

export default KanbanBoard;
```

### Admin Dashboard with AI Analytics

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [aiInsights, setAiInsights] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
    fetchAIInsights();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/admin/analytics`);
      setAnalytics(res.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchAIInsights = async () => {
    try {
      // Get burnout analysis for all users
      const usersRes = await axios.get(`${process.env.REACT_APP_API_URL}/users`);
      
      const insights = await Promise.all(
        usersRes.data.map(async (user) => {
          try {
            const burnoutRes = await axios.post(
              `${process.env.REACT_APP_ML_URL}/detect-burnout`,
              {
                user_id: user._id,
                tasks_assigned: user.tasksAssigned || 0,
                tasks_completed: user.tasksCompleted || 0,
                avg_task_duration: user.avgTaskDuration || 0,
                overtime_hours: user.overtimeHours || 0,
                missed_deadlines: user.missedDeadlines || 0
              }
            );
            return {
              user: user.username,
              ...burnoutRes.data
            };
          } catch (error) {
            return null;
          }
        })
      );

      setAiInsights(insights.filter(i => i && i.burnout_level !== 'low'));
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <StatCard title="Total Users" value={analytics?.totalUsers || 0} />
        <StatCard title="Active Tasks" value={analytics?.activeTasks || 0} />
        <StatCard title="Open Tickets" value={analytics?.openTickets || 0} />
        <StatCard title="Completion Rate" value={`${analytics?.completionRate || 0}%`} />
      </div>

      <div className="ai-insights-section">
        <h2>🤖 AI-Powered Insights</h2>
        {aiInsights.length === 0 ? (
          <p>No critical insights at this time</p>
        ) : (
          <div className="insights-list">
            {aiInsights.map((insight, idx) => (
              <div key={idx} className={`insight-card ${insight.burnout_level}`}>
                <h3>{insight.user}</h3>
                <div className="burnout-score">
                  Burnout Risk: {(insight.burnout_score * 100).toFixed(0
