---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with burnout detection"
  - "create user dashboard with kanban board"
  - "add AI ticket classification system"
  - "build admin panel with role-based access"
  - "configure ML service for risk prediction"
  - "deploy user management with AI insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user/task management, Kanban boards, time tracking, support ticket systems, and ML-based insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What It Does

- **User Management**: Secure authentication (JWT), role-based access control, user CRUD operations
- **Task Tracking**: Kanban board workflow, time tracking, task assignment and monitoring
- **Support System**: Ticket management with AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, security alerts

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User interface on port 3000
2. **Backend** (Node.js) - REST API on port 5000
3. **ML Service** (FastAPI + scikit-learn) - AI analytics on port 8000

## Installation

### Prerequisites

```bash
node -v  # v14+ required
python --version  # 3.8+ required
mongo --version  # MongoDB required
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
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const register = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// Login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};

// Make authenticated request
const getProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Create user
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority || 'medium',
      status: 'todo',
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};

// Track time on task
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return response.json();
};
```

### Ticket Management

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority || 'medium',
      category: ticketData.category
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status }) // 'open', 'in-progress', 'resolved', 'closed'
  });
  return response.json();
};
```

## ML Service API Usage

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        failed_login_attempts: 3,
        unusual_activity_hours: 2,
        data_access_volume: 150,
        policy_violations: 1,
        task_completion_rate: 0.85
      }
    })
  });
  return response.json();
  // Returns: { risk_score: 0.65, risk_level: 'medium', factors: [...] }
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userData.userId,
      activity_patterns: {
        login_time: '03:00',
        login_location: 'unusual_ip',
        actions_per_hour: 500,
        data_downloaded_mb: 2000
      }
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, alert_level: 'high' }
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_data: {
        weekly_hours: 55,
        tasks_completed: 45,
        tasks_overdue: 8,
        avg_task_complexity: 7.5,
        weekend_work_hours: 12,
        consecutive_work_days: 14
      }
    })
  });
  return response.json();
  // Returns: { burnout_risk: 0.78, risk_level: 'high', recommendations: [...] }
};
```

### Project Insights

```javascript
// Get predictive project insights
const getProjectInsights = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      metrics: {
        total_tasks: 100,
        completed_tasks: 60,
        overdue_tasks: 15,
        team_size: 8,
        avg_completion_time_days: 5.2,
        deadline_days_remaining: 30
      }
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.35, estimated_completion_date: '2026-05-15', bottlenecks: [...] }
};
```

### AI Ticket Classification

```javascript
// Classify and route support ticket
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', assigned_team: 'backend', confidence: 0.89 }
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      fetchUserProfile(token);
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUserProfile = async (token) => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/profile`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      console.error('Auth error:', error);
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (credentials) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={(id) => moveTask(id, 'todo')} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={(id) => moveTask(id, 'in-progress')} />
      <Column title="Done" tasks={tasks.done} onMove={(id) => moveTask(id, 'done')} />
    </div>
  );
};

const Column = ({ title, tasks, onMove }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <TaskCard key={task._id} task={task} onMove={onMove} />
    ))}
  </div>
);

const TaskCard = ({ task, onMove }) => (
  <div className="task-card" draggable onDragEnd={() => onMove(task._id)}>
    <h4>{task.title}</h4>
    <p>{task.description}</p>
    <span className={`priority ${task.priority}`}>{task.priority}</span>
  </div>
);

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskResponse = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const riskData = await riskResponse.json();

      // Fetch burnout analysis
      const burnoutResponse = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/burnout-analysis`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const burnoutData = await burnoutResponse.json();

      setAnalytics({
        riskScore: riskData.risk_score,
        burnoutRisk: burnoutData.burnout_risk,
        anomalies: riskData.factors || []
      });
    } catch (error) {
      console.error('Analytics fetch error:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${getRiskLevel(analytics.riskScore)}`}>
          {(analytics.riskScore * 100).toFixed(0)}%
        </div>
      </div>
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${getRiskLevel(analytics.burnoutRisk)}`}>
          {(analytics.burnoutRisk * 100).toFixed(0)}%
        </div>
      </div>
      <div className="anomalies">
        <h3>Detected Anomalies</h3>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="anomaly-item">{anomaly}</div>
        ))}
      </div>
    </div>
  );
};

const getRiskLevel = (score) => {
  if (score < 0.3) return 'low';
  if (score < 0.7) return 'medium';
  return 'high';
};

export default AIAnalytics;
```

## Configuration

### Backend Configuration

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
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

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    model_path: str = os.getenv('MODEL_PATH', './models')
    backend_url: str = os.getenv('BACKEND_URL', 'http://localhost:5000')
    log_level: str = os.getenv('LOG_LEVEL', 'INFO')
    
    class Config:
        env_file = '.env'

settings = Settings()
```

```python
# ml-service/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI(title="Enterprise UMS ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    user_id: str
    features: dict

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    workload_data: dict

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    # Extract features
    features = [
        request.features.get('failed_login_attempts', 0),
        request.features.get('unusual_activity_hours', 0),
        request.features.get('data_access_volume', 0),
        request.features.get('policy_violations', 0),
        request.features.get('task_completion_rate', 1.0)
    ]
    
    # Simple risk calculation (replace with trained model)
    risk_score = np.clip(sum([
        features[0] * 0.2,
        features[1] * 0.15,
        features[2] / 1000 * 0.3,
        features[3] * 0.25,
        (1 - features[4]) * 0.1
    ]), 0, 1)
    
    risk_level = 'low' if risk_score < 0.3 else 'medium' if risk_score < 0.7 else 'high'
    
    factors = []
    if features[0] > 2:
        factors.append('Multiple failed login attempts')
    if features[1] > 1:
        factors.append('Unusual activity hours')
    if features[3] > 0:
        factors.append('Policy violations detected')
    
    return {
        'risk_score': float(risk_score),
        'risk_level': risk_level,
        'factors': factors
    }

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    data = request.workload_data
    
    # Calculate burnout risk
    weekly_hours_score = min(data.get('weekly_hours', 40) / 60, 1.0)
    overdue_score = min(data.get('tasks_overdue', 0) / 10, 1.0)
    weekend_score = min(data.get('weekend_work_hours', 0) / 20, 1.0)
    consecutive_days_score = min(data.get('consecutive_work_days', 5) / 30, 1.0)
    
    burnout_risk = np.clip(
        weekly_hours_score * 0.3 + 
        overdue_score * 0.2 + 
        weekend_score * 0.3 + 
        consecutive_days_score * 0.2,
        0, 1
    )
    
    risk_level = 'low' if burnout_risk < 0.3 else 'medium' if burnout_risk < 0.7 else 'high'
    
    recommendations = []
    if weekly_hours_score > 0.7:
        recommendations.append('Reduce weekly working hours')
    if weekend_score > 0.5:
        recommendations.append('Avoid weekend work')
    if consecutive_days_score > 0.6:
        recommendations.append('Take regular breaks')
    
    return {
        'burnout_risk': float(burnout_risk),
        'risk_level': risk_level,
        'recommendations': recommendations
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracker Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
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

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  const handleSave = async () => {
    const minutes = Math.floor(seconds / 60);
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeSpent: minutes })
    });
    setSeconds(0);
    setIsRunning(false);
    onSave && onSave(minutes);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave} disabled={seconds === 0}>
        Save Time
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string in .env
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
```

### JWT Token Errors

```javascript
// Verify token is being sent correctly
const token = localStorage.getItem('token');
console.log('Token:', token);

// Check token expiry
const decoded = jwt.decode(token);
console.log('Token expires:', new Date(decoded.exp * 1000));

// Clear expired token
if (Date.now() >= decoded.exp * 1000) {
  localStorage.removeItem('token');
  window.location.href = '/login';
}
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

# Check logs
tail -f ml-service/logs/app.log

# Reinstall dependencies
pip install -r requirements.txt --force-reinstall

# Restart with verbose logging
uvicorn main:app --reload --log-level debug
```

### Port Already in Use

```bash
# Find process using port
lsof -i :5000
lsof -i :3000
lsof -i :8000

# Kill process
kill -9 <PID>

# Or use different ports in .env files
```

### Database Schema Issues

```javascript
// backend/scripts/seedData.js
const mongoose = require('mongoose');
require('dotenv').config();

const seedDatabase = async () => {
  await mongoose.connect(process
