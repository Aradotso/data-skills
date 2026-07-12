---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and ticket classification
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management system with task tracking"
  - "implement AI-powered ticket classification system"
  - "build admin dashboard with user analytics"
  - "add burnout detection to user management"
  - "configure kanban board with time tracking"
  - "integrate AI risk prediction for users"
  - "deploy enterprise user management platform"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to work with the Enterprise User Management System with AI Analytics, a full-stack JavaScript application combining React frontend, Node.js backend, and FastAPI ML services for intelligent user, task, and ticket management with AI-powered insights.

## What This Project Does

The Enterprise User Management System provides:
- **User & Admin Management**: JWT-authenticated role-based access control
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Dashboards**: Real-time analytics for administrators and performance tracking for users

## Installation & Setup

### Prerequisites
```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Install

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Edit .env with your configuration

# Frontend setup
cd ../frontend
npm install
cp .env.example .env

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env
```

### Environment Configuration

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env)**:
```bash
PORT=8000
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
```

## Running the Application

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - Frontend
cd frontend
npm start

# Terminal 3 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000
```

## Key API Endpoints

### Authentication
```javascript
// Login
POST /api/auth/login
{
  "email": "admin@company.com",
  "password": "password123"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass",
  "role": "user"
}
```

### User Management
```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: Bearer ${token} }

// Update user
PUT /api/users/:id
{
  "name": "Updated Name",
  "role": "admin",
  "department": "Engineering"
}

// Delete user
DELETE /api/users/:id
```

### Task Management
```javascript
// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Ticket Management
```javascript
// Create support ticket
POST /api/tickets
{
  "title": "Login issues",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open&priority=high
```

### AI Analytics Endpoints
```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "123",
  "loginFrequency": 15,
  "taskCompletionRate": 0.45,
  "failedLoginAttempts": 3
}

// Burnout detection
POST /api/ml/detect-burnout
{
  "userId": "123",
  "weeklyHours": 65,
  "tasksAssigned": 25,
  "tasksCompleted": 8
}

// Ticket classification
POST /api/ml/classify-ticket
{
  "title": "Cannot reset password",
  "description": "The reset link doesn't work"
}
```

## Frontend Integration Patterns

### Authentication Context
```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

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
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (err) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
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
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`);
      const grouped = res.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (err) {
      console.error('Failed to fetch tasks:', err);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (err) {
      console.error('Failed to update task:', err);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    moveTask(taskId, status);
  };

  const allowDrop = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={allowDrop}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component
```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  const saveTime = async () => {
    try {
      await axios.post(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
        duration: time
      });
      setTime(0);
      setIsRunning(false);
    } catch (err) {
      console.error('Failed to save time:', err);
    }
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(time)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTime} disabled={time === 0}>
        Save
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Backend API Implementation

### User Controller
```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    const user = new User({ name, email, password, role, department });
    await user.save();

    res.status(201).json({
      _id: user._id,
      name: user.name,
      email: user.email,
      role: user.role
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    // Don't allow password updates through this endpoint
    delete updates.password;

    const user = await User.findByIdAndUpdate(
      id,
      { $set: updates },
      { new: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};
```

### Task Controller
```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate, status } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user._id,
      priority,
      dueDate,
      status: status || 'todo'
    });

    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json(task);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findByIdAndUpdate(
      id,
      { 
        status,
        ...(status === 'done' ? { completedAt: new Date() } : {})
      },
      { new: true }
    ).populate('assignedTo', 'name email');

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    res.json(task);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.addTimeLog = async (req, res) => {
  try {
    const { id } = req.params;
    const { duration } = req.body;

    const task = await Task.findByIdAndUpdate(
      id,
      { 
        $push: { 
          timeLogs: { 
            user: req.user._id, 
            duration, 
            timestamp: new Date() 
          } 
        }
      },
      { new: true }
    );

    res.json(task);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};
```

## ML Service Implementation

### AI Analytics Service
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from river import tree, ensemble
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskPredictionInput(BaseModel):
    userId: str
    loginFrequency: int
    taskCompletionRate: float
    failedLoginAttempts: int
    avgResponseTime: float = 0.0

class BurnoutDetectionInput(BaseModel):
    userId: str
    weeklyHours: float
    tasksAssigned: int
    tasksCompleted: int
    missedDeadlines: int = 0

class TicketClassificationInput(BaseModel):
    title: str
    description: str

# Initialize online learning model for risk prediction
risk_model = ensemble.AdaptiveRandomForestClassifier(
    n_models=10,
    seed=42
)

@app.post("/predict-risk")
async def predict_risk(data: RiskPredictionInput):
    try:
        # Feature engineering
        features = {
            'login_frequency': data.loginFrequency,
            'task_completion_rate': data.taskCompletionRate,
            'failed_logins': data.failedLoginAttempts,
            'avg_response': data.avgResponseTime
        }
        
        # Calculate risk score
        risk_score = 0
        if data.failedLoginAttempts > 5:
            risk_score += 0.4
        if data.taskCompletionRate < 0.5:
            risk_score += 0.3
        if data.loginFrequency > 50:
            risk_score += 0.2
        if data.avgResponseTime > 5.0:
            risk_score += 0.1
            
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "userId": data.userId,
            "riskScore": round(risk_score, 2),
            "riskLevel": risk_level,
            "factors": features,
            "recommendations": get_risk_recommendations(risk_level, features)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(data: BurnoutDetectionInput):
    try:
        completion_rate = data.tasksCompleted / data.tasksAssigned if data.tasksAssigned > 0 else 0
        
        burnout_score = 0
        if data.weeklyHours > 50:
            burnout_score += 0.3
        if completion_rate < 0.6:
            burnout_score += 0.3
        if data.missedDeadlines > 3:
            burnout_score += 0.2
        if data.tasksAssigned > 20:
            burnout_score += 0.2
            
        burnout_level = "critical" if burnout_score > 0.7 else "warning" if burnout_score > 0.4 else "normal"
        
        return {
            "userId": data.userId,
            "burnoutScore": round(burnout_score, 2),
            "burnoutLevel": burnout_level,
            "weeklyHours": data.weeklyHours,
            "completionRate": round(completion_rate, 2),
            "recommendations": get_burnout_recommendations(burnout_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(data: TicketClassificationInput):
    try:
        text = f"{data.title} {data.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["bug", "error", "crash", "not working", "broken"],
            "account": ["login", "password", "access", "authentication"],
            "feature": ["request", "enhancement", "suggestion", "improve"],
            "billing": ["payment", "invoice", "subscription", "charge"]
        }
        
        category = "general"
        max_matches = 0
        
        for cat, keywords in categories.items():
            matches = sum(1 for keyword in keywords if keyword in text)
            if matches > max_matches:
                max_matches = matches
                category = cat
                
        # Determine priority based on keywords
        urgent_keywords = ["critical", "urgent", "down", "crash", "emergency"]
        priority = "high" if any(word in text for word in urgent_keywords) else "medium"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": round(max_matches / 5, 2),
            "suggestedDepartment": get_department_for_category(category)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(risk_level, features):
    recommendations = []
    if risk_level == "high":
        recommendations.append("Immediate security review required")
        if features['failed_logins'] > 5:
            recommendations.append("Reset password and enable 2FA")
    if features['task_completion_rate'] < 0.5:
        recommendations.append("Review workload distribution")
    return recommendations

def get_burnout_recommendations(burnout_level):
    if burnout_level == "critical":
        return ["Reduce workload immediately", "Schedule time off", "Reassign tasks"]
    elif burnout_level == "warning":
        return ["Monitor closely", "Consider workload reduction"]
    return ["Maintain current balance"]

def get_department_for_category(category):
    mapping = {
        "technical": "Engineering",
        "account": "IT Support",
        "feature": "Product",
        "billing": "Finance"
    }
    return mapping.get(category, "General Support")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Common Patterns & Best Practices

### Protected Route Middleware
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ error: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Token invalid' });
  }
};

exports.restrictTo = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    next();
  };
};
```

### Admin Dashboard Analytics
```javascript
// src/pages/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completedTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [users, tasks, tickets, risks] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/users`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/risks`)
      ]);

      setAnalytics({
        totalUsers: users.data.length,
        activeUsers: users.data.filter(u => u.isActive).length,
        totalTasks: tasks.data.length,
        completedTasks: tasks.data.filter(t => t.status === 'done').length,
        openTickets: tickets.data.filter(t => t.status === 'open').length,
        highRiskUsers: risks.data.filter(r => r.riskLevel === 'high')
      });
    } catch (err) {
      console.error('Failed to fetch analytics:', err);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p className="metric-value">{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Users</h3>
          <p className="metric-value">{analytics.activeUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Task Completion</h3>
          <p className="metric-value">
            {analytics.totalTasks > 0 
              ? `${Math.round((analytics.completedTasks / analytics.totalTasks) * 100)}%`
              : '0%'}
          </p>
        </div>
        <div className="metric-card alert">
          <h3>High Risk Users</h3>
          <p className="metric-value">{analytics.highRiskUsers.length}</p>
        </div>
      </div>
      
      {analytics.highRiskUsers.length > 0 && (
        <div className="alerts-section">
          <h2>Security Alerts</h2>
          {analytics.highRiskUsers.map(user => (
            <div key={user.userId} className="alert-item">
              <span>{user.userName}</span>
              <span className="risk-badge high">{user.riskLevel}</span>
              <button onClick={() => handleUserReview(user.userId)}>
                Review
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

## Troubleshooting

### JWT Token Issues
```javascript
// If tokens expire too quickly, adjust in backend/.env
JWT_EXPIRE=7d  // Increase duration

// Clear invalid tokens in frontend
localStorage.removeItem('token');
delete axios.defaults.headers.common['Authorization'];
```

### MongoDB Connection Errors
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB Connected');
  } catch (err) {
    console.error('MongoDB connection error:', err);
    process.exit(1);
  }
};

module.exports = connectDB;
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
# Check if service is running
curl http://localhost:8000/health

# Restart with verbose logging
uvicorn main:app --reload --log-level debug

# Check Python dependencies
pip list | grep -E "fastapi|uvicorn|scikit-learn|river"
```

### Task Status Not Updating
```javascript
// Ensure proper status values
const validStatuses = ['todo', 'in-progress', 'done'];

// Add validation in backend
if (!validStatuses.includes(status)) {
  return res.status(400).json({ error: 'Invalid status' });
}
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to assist developers in implementing, customizing, and troubleshooting the application.
