---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and ticket classification
triggers:
  - how do I set up the enterprise user management system with AI analytics
  - integrate AI-powered user management into my application
  - implement risk detection and anomaly detection for user behavior
  - create a user management system with ticket classification
  - build an admin dashboard with AI insights
  - add burnout detection to user management
  - set up JWT authentication with role-based access control
  - implement kanban board with time tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: JWT authentication, role-based access control (Admin/User)
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket creation, tracking, and AI-powered classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **ML Service**: FastAPI backend using scikit-learn and River for online learning

The system consists of three components:
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js REST API (port 5000)
- **ML Service**: FastAPI Python service (port 8000)

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB (local or cloud)
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env  # Configure environment variables

# Setup ML service
cd ../ml-service
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Terminal 3: Frontend
cd frontend
npm start
```

### Development Mode

```bash
# Backend with nodemon
cd backend
npm run dev

# ML Service with auto-reload
cd ml-service
uvicorn main:app --reload

# Frontend
cd frontend
npm start
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return await response.json();
};

// Login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};

// Protected request with JWT
const getProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/auth/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### User Management (Admin Only)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
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
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'in-progress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};

// Track time on task
const trackTaskTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```python
# Python client example
import requests

def classify_ticket(ticket_text, priority):
    """Classify support ticket using AI"""
    response = requests.post(
        'http://localhost:8000/api/ml/classify-ticket',
        json={
            'text': ticket_text,
            'priority': priority
        }
    )
    return response.json()

# Example usage
result = classify_ticket(
    "Cannot access database, getting connection timeout errors",
    "high"
)
print(f"Category: {result['category']}")
print(f"Confidence: {result['confidence']}")
print(f"Suggested routing: {result['routing']}")
```

```javascript
// JavaScript client example
const classifyTicket = async (ticketText, priority) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      priority: priority
    })
  });
  return await response.json();
};

// Use in ticket creation
const ticket = await classifyTicket(
  "Login page not loading, shows 404 error",
  "medium"
);
console.log('Category:', ticket.category); // e.g., "technical"
console.log('Should route to:', ticket.routing); // e.g., "engineering_team"
```

### Risk Detection

```javascript
// Detect user risk based on behavior
const detectUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      metrics: {
        loginAttempts: 5,
        failedLogins: 3,
        unusualAccessTime: true,
        dataAccessPattern: "abnormal",
        multipleIPs: true
      }
    })
  });
  const result = await response.json();
  return result;
};

// Example response
{
  "riskScore": 0.78,
  "riskLevel": "high",
  "factors": ["multiple_failed_logins", "unusual_access_pattern"],
  "recommendation": "require_mfa"
}
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (userActivityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivityData.userId,
      activities: userActivityData.activities,
      timeWindow: "24h"
    })
  });
  return await response.json();
};

// Example usage
const anomalies = await detectAnomalies({
  userId: "user123",
  activities: [
    { action: "data_export", timestamp: "2024-01-15T02:30:00Z", volume: "500MB" },
    { action: "login", timestamp: "2024-01-15T03:00:00Z", location: "foreign_country" }
  ]
});

console.log('Is anomalous:', anomalies.isAnomaly);
console.log('Anomaly score:', anomalies.score);
```

### Burnout Detection

```javascript
// Analyze user workload for burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      workloadMetrics: {
        tasksCompleted: 45,
        averageHoursPerDay: 11,
        weekendsWorked: 3,
        overdueTasksPercent: 35,
        avgTaskCompletionTime: 150 // minutes
      }
    })
  });
  return await response.json();
};

// Example response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.82,
  "recommendations": [
    "reduce_workload",
    "schedule_time_off",
    "redistribute_tasks"
  ],
  "workloadAnalysis": {
    "overworkedDays": 18,
    "taskBalanceScore": 0.35
  }
}
```

### Project Delay Prediction

```javascript
// Predict project completion delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      metrics: {
        totalTasks: projectData.totalTasks,
        completedTasks: projectData.completedTasks,
        avgCompletionRate: projectData.avgCompletionRate,
        teamSize: projectData.teamSize,
        daysRemaining: projectData.daysRemaining,
        complexityScore: projectData.complexityScore
      }
    })
  });
  return await response.json();
};

// Example usage
const prediction = await predictProjectDelay({
  id: "proj-2024-01",
  totalTasks: 100,
  completedTasks: 45,
  avgCompletionRate: 3, // tasks per day
  teamSize: 5,
  daysRemaining: 15,
  complexityScore: 0.7
});

console.log('Delay probability:', prediction.delayProbability);
console.log('Expected delay days:', prediction.expectedDelayDays);
console.log('Recommendations:', prediction.recommendations);
```

## React Component Patterns

### Authentication Context

```javascript
// AuthContext.js
import React, { createContext, useState, useContext, useEffect } from 'react';

const AuthContext = createContext();

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
      const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/profile`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Failed to fetch profile:', error);
      localStorage.removeItem('token');
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
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
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { useAuth } from './AuthContext';

const KanbanBoard = () => {
  const { user } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
// AdminDashboard.js
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchAIAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/admin/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };

  const fetchAIAlerts = async () => {
    const token = localStorage.getItem('token');
    
    // Get risk alerts
    const riskResponse = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/alerts/risk`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const riskData = await riskResponse.json();
    
    // Get burnout alerts
    const burnoutResponse = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/alerts/burnout`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const burnoutData = await burnoutResponse.json();
    
    setAlerts([...riskData.alerts, ...burnoutData.alerts]);
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
      
      {analytics && (
        <div className="analytics-grid">
          <div className="metric-card">
            <h3>Total Users</h3>
            <p className="metric-value">{analytics.totalUsers}</p>
          </div>
          <div className="metric-card">
            <h3>Active Tasks</h3>
            <p className="metric-value">{analytics.activeTasks}</p>
          </div>
          <div className="metric-card">
            <h3>Pending Tickets</h3>
            <p className="metric-value">{analytics.pendingTickets}</p>
          </div>
        </div>
      )}

      <div className="ai-alerts">
        <h3>AI-Powered Alerts</h3>
        {alerts.map((alert, idx) => (
          <div key={idx} className={`alert alert-${alert.severity}`}>
            <h4>{alert.type}</h4>
            <p>{alert.message}</p>
            <span>{alert.user}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Configuration

```javascript
// config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

```javascript
// middleware/auth.js
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

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### ML Service Configuration

```python
# config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    model_path: str = os.getenv("MODEL_PATH", "./models")
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    backend_url: str = os.getenv("BACKEND_URL", "http://localhost:5000")
    
    class Config:
        env_file = ".env"

settings = Settings()
```

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
risk_model = None
burnout_model = None

@app.on_event("startup")
async def load_models():
    global risk_model, burnout_model
    try:
        risk_model = joblib.load("./models/risk_model.pkl")
        burnout_model = joblib.load("./models/burnout_model.pkl")
    except FileNotFoundError:
        print("Models not found, training new ones...")

class TicketClassificationRequest(BaseModel):
    text: str
    priority: str

class RiskDetectionRequest(BaseModel):
    userId: str
    metrics: dict

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    # Implement classification logic
    categories = ["technical", "billing", "account", "feature_request"]
    # Simplified example
    if "error" in request.text.lower() or "bug" in request.text.lower():
        category = "technical"
    elif "payment" in request.text.lower() or "invoice" in request.text.lower():
        category = "billing"
    else:
        category = "account"
    
    return {
        "category": category,
        "confidence": 0.85,
        "routing": f"{category}_team"
    }

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    metrics = request.metrics
    
    # Calculate risk score
    risk_score = 0.0
    factors = []
    
    if metrics.get("failedLogins", 0) > 2:
        risk_score += 0.3
        factors.append("multiple_failed_logins")
    
    if metrics.get("unusualAccessTime"):
        risk_score += 0.2
        factors.append("unusual_access_pattern")
    
    if metrics.get("multipleIPs"):
        risk_score += 0.25
        factors.append("multiple_ip_addresses")
    
    risk_level = "high" if risk_score > 0.6 else "medium" if risk_score > 0.3 else "low"
    
    return {
        "riskScore": risk_score,
        "riskLevel": risk_level,
        "factors": factors,
        "recommendation": "require_mfa" if risk_level == "high" else "monitor"
    }

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: dict):
    metrics = request["workloadMetrics"]
    
    # Calculate burnout score
    burnout_score = 0.0
    
    if metrics["averageHoursPerDay"] > 10:
        burnout_score += 0.3
    if metrics["weekendsWorked"] > 1:
        burnout_score += 0.25
    if metrics["overdueTasksPercent"] > 30:
        burnout_score += 0.25
    
    risk_level = "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low"
    
    recommendations = []
    if burnout_score > 0.6:
        recommendations = ["reduce_workload", "schedule_time_off", "redistribute_tasks"]
    
    return {
        "burnoutRisk": risk_level,
        "burnoutScore": burnout_score,
        "recommendations": recommendations,
        "workloadAnalysis": {
            "overworkedDays": metrics.get("weekendsWorked", 0) * 2 + 5,
            "taskBalanceScore": 1 - burnout_score
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Patterns

### Protected Routes

```javascript
// ProtectedRoute.js
import { Navigate } from 'react-router-dom';
import { useAuth } from './AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Real-time Notifications

```javascript
// NotificationService.js
import { useEffect } from 'react';
import io from 'socket.io-client';

export const useNotifications = (userId) => {
  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL, {
      auth: { token: localStorage.getItem('token') }
    });

    socket.on('notification', (data) => {
      // Show notification
      if (Notification.permission === 'granted') {
        new Notification(data.title, {
          body: data.message,
          icon: '/logo.png'
        });
      }
    });

    return () => socket.disconnect();
  }, [userId]);
};
```

### API Error Handling

```javascript
// api/client.js
const apiClient = {
  async request(endpoint, options = {}) {
    const token = localStorage.getItem('token');
    const config = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
        ...(token && { 'Authorization': `Bearer ${token}` })
      }
    };

    try {
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}${endpoint}`,
        config
      );

      if (response.status === 401) {
        localStorage.removeItem('token');
        window.location.href = '/login';
        throw new Error('Unauthorized');
      }

      const data = await response.json();
      
      if (!response.ok) {
        throw new Error(data.message || 'Request failed');
      }

      return data;
    } catch (error) {
      console.error('API Error:', error);
      throw error;
    }
  },

  get(endpoint) {
    return this.request(endpoint);
  },

  post(endpoint, data) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  },

  put(endpoint, data) {
    return this.request(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  },

  delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }
};

export default apiClient;
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
mongod --version
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string
# Ensure MONGODB_URI in .env is correct
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

### JWT
