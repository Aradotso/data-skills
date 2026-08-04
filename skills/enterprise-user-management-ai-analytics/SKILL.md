---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement role-based access control with AI"
  - "create user dashboard with task tracking"
  - "integrate AI ticket classification system"
  - "build admin panel with user analytics"
  - "deploy enterprise management system with ML"
  - "configure JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack enterprise user management platform that combines traditional CRUD operations with AI-powered analytics. The system features role-based access control, task management with Kanban boards, support ticket handling, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

## What This Project Does

This system provides three integrated components:

1. **Frontend (React)**: User and admin dashboards with task management, ticket tracking, and analytics visualization
2. **Backend (Node.js)**: REST API with JWT authentication, user management, and business logic
3. **ML Service (FastAPI)**: AI-powered analytics for ticket classification, risk detection, and predictive insights

**Key Capabilities:**
- Secure user authentication and authorization
- Task management with Kanban workflow
- Support ticket system with AI classification
- Real-time analytics and performance tracking
- AI-based risk and anomaly detection
- Burnout prediction using workload analysis
- Predictive project delay detection

## Installation

### Prerequisites

```bash
# Node.js 14+ for backend and frontend
node --version

# Python 3.8+ for ML service
python --version

# MongoDB running locally or connection string
```

### Clone and Setup

```bash
# Clone repository
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
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

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
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    })
  });
  const data = await response.json();
  return data; // { token, user }
};

// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data;
};

// GET /api/auth/me
const getCurrentUser = async (token) => {
  const response = await fetch('http://localhost:5000/api/auth/me', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### User Management (Admin Only)

```javascript
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'inProgress', 'done'
    })
  });
  return await response.json();
};

// GET /api/tasks - Get tasks (filtered by user if not admin)
const getTasks = async (token, filters = {}) => {
  const queryString = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tasks?${queryString}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PUT /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// POST /api/tasks/:id/time - Track time on task
const trackTaskTime = async (taskId, timeSpent, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, status = null) => {
  const url = status 
    ? `http://localhost:5000/api/tickets?status=${status}`
    : 'http://localhost:5000/api/tickets';
  const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// PUT /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

## ML Service API Reference

### AI Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  return await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
};
```

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const detectUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ userId })
  });
  return await response.json();
  // Returns: { riskScore: 0.65, riskLevel: 'medium', factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomalies = async (userId, activityData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      userId,
      loginTime: activityData.loginTime,
      activityCount: activityData.activityCount,
      location: activityData.location
    })
  });
  return await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, reason: 'Unusual login time' }
};
```

### Burnout Analysis

```javascript
// POST /api/ml/burnout-analysis
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ userId })
  });
  return await response.json();
  // Returns: { burnoutScore: 0.73, level: 'high', recommendations: [...] }
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-prediction
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      projectId: projectData.id,
      tasksCompleted: projectData.completed,
      tasksTotal: projectData.total,
      daysRemaining: projectData.daysLeft,
      teamSize: projectData.teamSize
    })
  });
  return await response.json();
  // Returns: { delayProbability: 0.68, estimatedDelay: 5, suggestions: [...] }
};
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchCurrentUser = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'inProgress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PUT',
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
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select
                value={task.status}
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inProgress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Insights Dashboard

```javascript
// src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AIInsights = ({ userId }) => {
  const { token } = useAuth();
  const [insights, setInsights] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });

  useEffect(() => {
    fetchInsights();
  }, [userId]);

  const fetchInsights = async () => {
    try {
      // Risk detection
      const riskRes = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({ userId })
      });
      const riskData = await riskRes.json();

      // Burnout analysis
      const burnoutRes = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({ userId })
      });
      const burnoutData = await burnoutRes.json();

      setInsights({
        risk: riskData,
        burnout: burnoutData,
        anomalies: []
      });
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    }
  };

  return (
    <div className="ai-insights">
      <h2>AI Analytics</h2>
      
      {insights.risk && (
        <div className={`insight-card risk-${insights.risk.riskLevel}`}>
          <h3>Risk Assessment</h3>
          <p>Risk Level: {insights.risk.riskLevel}</p>
          <p>Score: {(insights.risk.riskScore * 100).toFixed(1)}%</p>
          <ul>
            {insights.risk.factors?.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {insights.burnout && (
        <div className={`insight-card burnout-${insights.burnout.level}`}>
          <h3>Burnout Analysis</h3>
          <p>Level: {insights.burnout.level}</p>
          <p>Score: {(insights.burnout.burnoutScore * 100).toFixed(1)}%</p>
          <ul>
            {insights.burnout.recommendations?.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIInsights;
```

## Configuration

### Backend Configuration

```javascript
// backend/config/db.js
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

// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.id;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
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
    mongodb_uri: str = os.getenv("MONGODB_URI", "mongodb://localhost:27017/enterprise-user-mgmt")
    model_path: str = os.getenv("MODEL_PATH", "./models")
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    
    class Config:
        env_file = ".env"

settings = Settings()

# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

class RiskDetectionRequest(BaseModel):
    userId: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket using ML model
    """
    # Simplified example - in production, load trained model
    text = f"{request.subject} {request.description}".lower()
    
    # Basic rule-based classification (replace with actual ML model)
    if any(word in text for word in ['error', 'bug', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text for word in ['payment', 'invoice', 'billing']):
        category = 'billing'
        priority = 'medium'
    else:
        category = 'general'
        priority = 'low'
    
    return TicketClassificationResponse(
        category=category,
        priority=priority,
        confidence=0.85
    )

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    """
    Analyze user behavior for risk assessment
    """
    # Simplified example - replace with actual model
    risk_score = np.random.uniform(0.3, 0.9)
    
    if risk_score > 0.7:
        risk_level = 'high'
    elif risk_score > 0.4:
        risk_level = 'medium'
    else:
        risk_level = 'low'
    
    return {
        "riskScore": risk_score,
        "riskLevel": risk_level,
        "factors": [
            "Unusual activity pattern detected",
            "Multiple failed login attempts",
            "Access from new location"
        ]
    }

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: RiskDetectionRequest):
    """
    Analyze user workload for burnout risk
    """
    burnout_score = np.random.uniform(0.2, 0.95)
    
    if burnout_score > 0.7:
        level = 'high'
        recommendations = [
            "Consider reducing task load",
            "Schedule time off",
            "Redistribute urgent tasks"
        ]
    elif burnout_score > 0.4:
        level = 'medium'
        recommendations = [
            "Monitor workload closely",
            "Encourage breaks",
            "Review task priorities"
        ]
    else:
        level = 'low'
        recommendations = ["Workload appears manageable"]
    
    return {
        "burnoutScore": burnout_score,
        "level": level,
        "recommendations": recommendations
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './pages/AdminDashboard';
import UserDashboard from './pages/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly={true}>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const TimeTracker = ({ taskId }) => {
  const { token } = useAuth();
  const [isTracking, setIsTracking] = useState(false);
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    let interval = null;
    if (isTracking) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (interval) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isTracking]);

  const handleStart = () => {
    setIsTracking(true);
  };

  const handleStop = async () => {
    setIsTracking(false);
    const minutes = Math.floor(seconds / 60);
    
    // Save time to backend
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeSpent: minutes })
    });
    
    setSeconds(0);
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isTracking ? (
          <button onClick={handleStart}>Start Timer</button>
        ) : (
          <button onClick={handleStop}>Stop & Save</button>
        )}
      </div>
    </div>
  );
};

export default TimeTracker;
```

### Admin User Management

```javascript
// src/pages/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AdminDashboard = () => {
  const { token } = useAuth();
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchUsers();
    fetchAnalytics();
  }, []);

  const fetchUsers = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setUsers(data);
  };

  const fetchAnalytics = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/admin/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}` }
      });
      fetchUsers();
    }
  };

  const handleUpdateRole = async (userId, newRole) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ role: newRole })
    });
    fetchUsers();
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="analytics
