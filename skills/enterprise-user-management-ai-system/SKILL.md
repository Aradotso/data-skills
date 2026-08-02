---
name: enterprise-user-management-ai-system
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management with AI"
  - "implement user management system with task tracking"
  - "create admin dashboard with AI analytics"
  - "build user management app with ticket system"
  - "add AI-based risk detection to user management"
  - "integrate ML service for user behavior analysis"
  - "configure Kanban board with time tracking"
  - "deploy enterprise user management system"
---

# Enterprise User Management AI System

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system featuring AI-powered analytics, task management with Kanban boards, support ticket handling, and intelligent insights including risk detection, anomaly detection, and burnout analysis.

## What It Does

This system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards (To Do → In Progress → Done) with time tracking
- **Support System**: Ticket creation, tracking, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Controls**: User CRUD operations, audit logs, organization analytics
- **Real-time Insights**: Performance metrics, workload analysis, suspicious activity alerts

## Installation

### Prerequisites

```bash
# Node.js 14+ for backend/frontend
# Python 3.8+ for ML service
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
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

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token, user: { id, name, email, role } }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:userId
{
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement new feature",
  "description": "Build user profile page",
  "assignedTo": "userId",
  "status": "todo", // todo, inprogress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "inprogress",
  "timeSpent": 3600 // seconds
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets (admin)
GET /api/tickets?status=open&priority=high

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset sent"
}
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ai/risk-prediction
{
  "userId": "user123",
  "taskLoad": 15,
  "overdueCount": 3,
  "avgCompletionTime": 72
}
// Returns: { riskLevel: "high", probability: 0.78, factors: [...] }

// Anomaly detection
POST /api/ai/anomaly-detection
{
  "userId": "user123",
  "loginTime": "2026-04-15T03:30:00Z",
  "location": "unusual-ip",
  "activityPattern": [...]
}
// Returns: { isAnomaly: true, score: 0.85, reason: "..." }

// Burnout analysis
POST /api/ai/burnout-analysis
{
  "userId": "user123",
  "weeklyHours": 65,
  "taskCount": 25,
  "overtimeFrequency": 0.8
}
// Returns: { burnoutRisk: "high", recommendation: "..." }

// Project delay prediction
POST /api/ai/project-prediction
{
  "projectId": "proj123",
  "tasksCompleted": 40,
  "tasksRemaining": 60,
  "averageVelocity": 8,
  "deadline": "2026-06-01"
}
// Returns: { delayProbability: 0.65, estimatedCompletion: "2026-06-15" }
```

## Frontend Integration Examples

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const res = await axios.get(`${API_URL}/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/auth/login`, { email, password });
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

  return { user, loading, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/tasks/user/${userId}`);
      const grouped = res.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, status }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">{new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
      <select 
        value={status} 
        onChange={(e) => updateTaskStatus(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="inprogress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <TaskCard key={task._id} task={task} status={status} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Risk Dashboard Component

```javascript
// components/AIRiskDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const AIRiskDashboard = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);

  const fetchAIAnalytics = async () => {
    try {
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${API_URL}/ai/risk-prediction`, { userId }),
        axios.post(`${API_URL}/ai/burnout-analysis`, { userId })
      ]);
      setRiskData(riskRes.data);
      setBurnoutData(burnoutRes.data);
    } catch (error) {
      console.error('Error fetching AI analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Analyzing data...</div>;

  return (
    <div className="ai-dashboard">
      <div className="risk-card">
        <h3>Risk Level</h3>
        <div className={`risk-indicator ${riskData.riskLevel}`}>
          {riskData.riskLevel.toUpperCase()}
        </div>
        <p>Probability: {(riskData.probability * 100).toFixed(1)}%</p>
        <ul>
          {riskData.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="burnout-card">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-indicator ${burnoutData.burnoutRisk}`}>
          {burnoutData.burnoutRisk.toUpperCase()} RISK
        </div>
        <p>{burnoutData.recommendation}</p>
      </div>
    </div>
  );
};

export default AIRiskDashboard;
```

## Backend Implementation Patterns

### User Controller

```javascript
// controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { name, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, status },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }
    res.json({ success: true, data: {} });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
};
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a title'],
    trim: true,
    maxlength: [100, 'Title cannot exceed 100 characters']
  },
  description: {
    type: String,
    required: [true, 'Please add a description']
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date,
    required: [true, 'Please add a due date']
  },
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  completedAt: Date
}, {
  timestamps: true
});

// Index for efficient queries
TaskSchema.index({ assignedTo: 1, status: 1 });

module.exports = mongoose.model('Task', TaskSchema);
```

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ success: false, error: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ success: false, error: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        success: false, 
        error: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

## ML Service Implementation

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class RiskPredictionRequest(BaseModel):
    userId: str
    taskLoad: int
    overdueCount: int
    avgCompletionTime: float

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    location: str
    activityPattern: List[float]

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    weeklyHours: float
    taskCount: int
    overtimeFrequency: float

@app.post("/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Feature engineering
        features = np.array([[
            request.taskLoad,
            request.overdueCount,
            request.avgCompletionTime,
            request.taskLoad * request.overdueCount  # interaction term
        ]])
        
        # Simple rule-based model (replace with trained model)
        risk_score = (
            request.taskLoad * 0.3 + 
            request.overdueCount * 0.5 + 
            (request.avgCompletionTime / 24) * 0.2
        )
        
        risk_level = "low"
        if risk_score > 15:
            risk_level = "high"
        elif risk_score > 8:
            risk_level = "medium"
        
        factors = []
        if request.taskLoad > 10:
            factors.append("High task load")
        if request.overdueCount > 2:
            factors.append("Multiple overdue tasks")
        if request.avgCompletionTime > 48:
            factors.append("Slow task completion")
        
        return {
            "riskLevel": risk_level,
            "probability": min(risk_score / 20, 1.0),
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        from datetime import datetime
        
        # Parse login time
        login_hour = datetime.fromisoformat(request.loginTime.replace('Z', '+00:00')).hour
        
        # Anomaly detection logic
        is_anomaly = False
        score = 0.0
        reason = ""
        
        # Check unusual login time
        if login_hour < 6 or login_hour > 22:
            is_anomaly = True
            score += 0.4
            reason = "Login at unusual hours"
        
        # Check unusual location
        if "unusual" in request.location.lower():
            is_anomaly = True
            score += 0.5
            reason += "; Unusual location detected"
        
        return {
            "isAnomaly": is_anomaly,
            "score": min(score, 1.0),
            "reason": reason if reason else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        # Burnout risk calculation
        burnout_score = (
            (request.weeklyHours - 40) * 0.4 +
            request.taskCount * 0.3 +
            request.overtimeFrequency * 30
        )
        
        risk_level = "low"
        recommendation = "Workload is manageable. Keep up the good work!"
        
        if burnout_score > 20:
            risk_level = "high"
            recommendation = "Critical: Immediate workload reduction needed. Consider redistributing tasks."
        elif burnout_score > 10:
            risk_level = "medium"
            recommendation = "Warning: Monitor workload closely. Consider taking breaks."
        
        return {
            "burnoutRisk": risk_level,
            "score": burnout_score,
            "recommendation": recommendation,
            "metrics": {
                "weeklyHours": request.weeklyHours,
                "taskCount": request.taskCount,
                "overtimeFrequency": request.overtimeFrequency
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Configuration

### Backend Environment Variables

```bash
# .env (backend)
PORT=5000
NODE_ENV=production
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### Frontend Environment Variables

```bash
# .env (frontend)
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENV=development
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    model_path: str = os.getenv("MODEL_PATH", "./models")
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    max_workers: int = 4
    
    class Config:
        env_file = ".env"

settings = Settings()
```

## Common Patterns

### Admin Dashboard Data Fetching

```javascript
// pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const AdminDashboard = () => {
  const [stats, setStats] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  useEffect(() => {
    fetchDashboardStats();
  }, []);

  const fetchDashboardStats = async () => {
    try {
      const [usersRes, tasksRes, ticketsRes, riskRes] = await Promise.all([
        axios.get(`${API_URL}/users/count`),
        axios.get(`${API_URL}/tasks/active/count`),
        axios.get(`${API_URL}/tickets?status=open`),
        axios.get(`${API_URL}/ai/high-risk-users`)
      ]);

      setStats({
        totalUsers: usersRes.data.count,
        activeTasks: tasksRes.data.count,
        openTickets: ticketsRes.data.count,
        highRiskUsers: riskRes.data.users
      });
    } catch (error) {
      console.error('Error fetching dashboard stats:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{stats.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{stats.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{stats.openTickets}</p>
        </div>
      </div>
      
      {stats.highRiskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>High Risk Users</h2>
          <ul>
            {stats.highRiskUsers.map(user => (
              <li key={user.id}>{user.name} - {user.riskReason}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

### Time Tracker Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(seconds => seconds + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const toggle = () => {
    setIsActive(!isActive);
  };

  const reset = () => {
    setSeconds(0);
    setIsActive(false);
  };

  const saveTime = async () => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
        timeSpent: seconds
      });
      reset();
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        <button onClick={toggle}>{isActive ? 'Pause' : 'Start'}</button>
        <button onClick={reset}>Reset</button>
        <button onClick={saveTime} disabled={seconds === 0}>Save</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    // Retry connection
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### JWT Token Expiration Handling

```javascript
// frontend/utils/axios.js
import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

axiosInstance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default axiosInstance;
```

### ML Service Not Responding

```bash
# Check if service is running
curl http://localhost:8000/health

# View logs
tail -f ml-service/logs/app.log

# Restart with debugging
cd ml-service
uvicorn main:app --reload --log-level debug
```

### Task Status Not Updating

```javascript
// Ensure proper state management
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    const response = await axios.patch(
      `${API_URL}/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${localStorage.getItem('token')}` } }
    );
    
    // Update local state
    setTasks(prevTasks => ({
      ...prevTasks,
      [newStatus]: [...prevTasks[newStatus], response.data.data]
    }));
  } catch (error) {
    console.error('Update failed:', error.response?.data || error.message);
  }
};
```

### Performance Optimization

```javascript
// Use pagination for large datasets
GET /api/users?page=1&limit=20

// Backend implementation
exports.getAllUsers = async (req, res) => {
  const page = parseInt(req.query.page, 10) || 1;
  const limit = parseInt(req.query.limit, 10) || 20;
  const startIndex = (page - 1) * limit;

  const users = await User.find()
    .select('-password')
    .skip(startIndex)
    .limit(limit);

  const total = await User.countDocuments();

  res.json({
    success: true,
    count: users.length,
    total,
    page,
    pages: Math.ceil(total / limit),
    data: users
  });
};
```

This enterprise user management system provides a complete solution for managing users, tasks, and support with AI-powered insights for better decision-making
