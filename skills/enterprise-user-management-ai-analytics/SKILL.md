---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, and risk detection
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement AI-powered ticket classification system"
  - "build task management with burnout detection"
  - "add risk prediction to user management app"
  - "configure AI analytics for enterprise system"
  - "integrate ML service with user management backend"
  - "setup kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. The system provides user/admin dashboards, task management with Kanban boards, support ticket handling, and intelligent features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## Architecture Overview

The system consists of three main components:

- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js/Express REST API (port 5000)
- **ML Service**: FastAPI-based AI/ML microservice (port 8000)

All services communicate via REST APIs with JWT authentication for security.

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB (local or Atlas)
mongod --version
```

### Quick Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install all dependencies
cd backend && npm install
cd ../frontend && npm install
cd ../ml-service && pip install -r requirements.txt
```

## Configuration

### Backend Configuration

Create `backend/.env`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### ML Service Configuration

Create `ml-service/.env`:

```env
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
```

### Frontend Configuration

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

### Start Backend

```bash
cd backend
npm start
# Development with auto-reload
npm run dev
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --port 8000
# Or with production settings
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

### Start Frontend

```bash
cd frontend
npm start
```

## Backend API Examples

### User Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return response.json();
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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### Task Management

```javascript
// Create task
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
      priority: 'high', // low, medium, high
      status: 'todo', // todo, in-progress, done
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};

// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Support Ticket Management

```javascript
// Create support ticket
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
      category: ticketData.category
    })
  });
  return response.json();
};

// Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## ML Service API Examples

### AI-Powered Ticket Classification

```python
# In your FastAPI ML service (main.py)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib

app = FastAPI()

class TicketInput(BaseModel):
    subject: str
    description: str

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """
    Classify support ticket into categories
    Returns: category, priority, estimated_resolution_time
    """
    text = f"{ticket.subject} {ticket.description}"
    
    # Load pre-trained model
    model = joblib.load('models/ticket_classifier.pkl')
    vectorizer = joblib.load('models/tfidf_vectorizer.pkl')
    
    # Predict
    features = vectorizer.transform([text])
    category = model.predict(features)[0]
    confidence = max(model.predict_proba(features)[0])
    
    # Determine priority
    priority = "high" if confidence > 0.8 else "medium"
    
    return {
        "category": category,
        "priority": priority,
        "confidence": float(confidence),
        "estimated_resolution_hours": 24 if priority == "high" else 48
    }
```

### Risk Prediction

```python
class UserBehavior(BaseModel):
    user_id: str
    failed_login_attempts: int
    unusual_access_times: int
    data_access_volume: int
    permission_changes: int

@app.post("/predict-risk")
async def predict_risk(behavior: UserBehavior):
    """
    Predict user risk score based on behavior patterns
    """
    import numpy as np
    from sklearn.ensemble import IsolationForest
    
    # Load model
    model = joblib.load('models/risk_detector.pkl')
    
    # Prepare features
    features = np.array([[
        behavior.failed_login_attempts,
        behavior.unusual_access_times,
        behavior.data_access_volume,
        behavior.permission_changes
    ]])
    
    # Predict (-1 for anomaly, 1 for normal)
    prediction = model.predict(features)[0]
    anomaly_score = model.score_samples(features)[0]
    
    risk_level = "high" if prediction == -1 else "low"
    
    return {
        "user_id": behavior.user_id,
        "risk_level": risk_level,
        "risk_score": float(abs(anomaly_score)),
        "alert": prediction == -1,
        "recommendations": get_risk_recommendations(risk_level)
    }

def get_risk_recommendations(risk_level):
    if risk_level == "high":
        return [
            "Review user activity logs",
            "Verify recent permission changes",
            "Consider temporary access restriction"
        ]
    return ["Continue normal monitoring"]
```

### Burnout Detection

```python
class WorkloadData(BaseModel):
    user_id: str
    hours_worked_week: float
    tasks_completed: int
    tasks_overdue: int
    avg_task_completion_days: float
    weekend_work_hours: float

@app.post("/detect-burnout")
async def detect_burnout(workload: WorkloadData):
    """
    Analyze workload patterns to detect potential burnout
    """
    # Burnout indicators
    burnout_score = 0
    factors = []
    
    if workload.hours_worked_week > 45:
        burnout_score += 30
        factors.append("excessive_hours")
    
    if workload.tasks_overdue > 5:
        burnout_score += 25
        factors.append("overdue_tasks")
    
    if workload.avg_task_completion_days > 7:
        burnout_score += 20
        factors.append("slow_completion")
    
    if workload.weekend_work_hours > 5:
        burnout_score += 25
        factors.append("weekend_work")
    
    risk_level = "high" if burnout_score > 50 else "medium" if burnout_score > 30 else "low"
    
    return {
        "user_id": workload.user_id,
        "burnout_score": burnout_score,
        "risk_level": risk_level,
        "contributing_factors": factors,
        "recommendations": get_burnout_recommendations(risk_level)
    }

def get_burnout_recommendations(risk_level):
    if risk_level == "high":
        return [
            "Redistribute workload immediately",
            "Schedule 1-on-1 with manager",
            "Consider time off",
            "Review task priorities"
        ]
    elif risk_level == "medium":
        return [
            "Monitor workload trends",
            "Encourage breaks",
            "Review task assignments"
        ]
    return ["Workload appears healthy"]
```

## Frontend Integration Examples

### React Hook for AI Features

```javascript
// hooks/useAIAnalytics.js
import { useState, useEffect } from 'react';

export const useAIAnalytics = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const classifyTicket = async (subject, description) => {
    setLoading(true);
    try {
      const response = await fetch('http://localhost:8000/classify-ticket', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ subject, description })
      });
      const data = await response.json();
      setLoading(false);
      return data;
    } catch (err) {
      setError(err.message);
      setLoading(false);
      throw err;
    }
  };
  
  const checkBurnout = async (userId) => {
    setLoading(true);
    try {
      // First get workload data from backend
      const workloadResponse = await fetch(
        `http://localhost:5000/api/analytics/workload/${userId}`,
        {
          headers: { 'Authorization': `Bearer ${localStorage.getItem('token')}` }
        }
      );
      const workloadData = await workloadResponse.json();
      
      // Send to ML service
      const mlResponse = await fetch('http://localhost:8000/detect-burnout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(workloadData)
      });
      const analysis = await mlResponse.json();
      setLoading(false);
      return analysis;
    } catch (err) {
      setError(err.message);
      setLoading(false);
      throw err;
    }
  };
  
  return { classifyTicket, checkBurnout, loading, error };
};
```

### Ticket Creation with AI Classification

```javascript
// components/CreateTicket.jsx
import React, { useState } from 'react';
import { useAIAnalytics } from '../hooks/useAIAnalytics';

const CreateTicket = () => {
  const [subject, setSubject] = useState('');
  const [description, setDescription] = useState('');
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const { classifyTicket, loading } = useAIAnalytics();
  
  const handleClassify = async () => {
    const result = await classifyTicket(subject, description);
    setAiSuggestion(result);
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    const token = localStorage.getItem('token');
    const ticketData = {
      subject,
      description,
      category: aiSuggestion?.category || 'general',
      priority: aiSuggestion?.priority || 'medium'
    };
    
    const response = await fetch('http://localhost:5000/api/tickets', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify(ticketData)
    });
    
    if (response.ok) {
      alert('Ticket created successfully!');
      setSubject('');
      setDescription('');
      setAiSuggestion(null);
    }
  };
  
  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          placeholder="Subject"
          value={subject}
          onChange={(e) => setSubject(e.target.value)}
          required
        />
        <textarea
          placeholder="Description"
          value={description}
          onChange={(e) => setDescription(e.target.value)}
          required
        />
        
        <button type="button" onClick={handleClassify} disabled={loading}>
          {loading ? 'Classifying...' : 'AI Classify'}
        </button>
        
        {aiSuggestion && (
          <div className="ai-suggestion">
            <h4>AI Suggestions:</h4>
            <p>Category: {aiSuggestion.category}</p>
            <p>Priority: {aiSuggestion.priority}</p>
            <p>Confidence: {(aiSuggestion.confidence * 100).toFixed(1)}%</p>
          </div>
        )}
        
        <button type="submit">Create Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

### Admin Dashboard with Analytics

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    loadDashboardData();
  }, []);
  
  const loadDashboardData = async () => {
    // Get analytics
    const analyticsRes = await fetch('http://localhost:5000/api/analytics/dashboard', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);
    
    // Get users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);
  };
  
  const checkUserRisk = async (userId) => {
    // Get user behavior data
    const behaviorRes = await fetch(`http://localhost:5000/api/analytics/behavior/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const behavior = await behaviorRes.json();
    
    // Check risk with ML service
    const riskRes = await fetch('http://localhost:8000/predict-risk', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(behavior)
    });
    const riskData = await riskRes.json();
    
    if (riskData.alert) {
      alert(`High risk detected for user ${userId}!\n${riskData.recommendations.join('\n')}`);
    }
  };
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="stats-grid">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
          <div className="stat-card">
            <h3>High Risk Alerts</h3>
            <p>{analytics.riskAlerts}</p>
          </div>
        </div>
      )}
      
      <div className="users-table">
        <h2>Users</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.isActive ? 'Active' : 'Inactive'}</td>
                <td>
                  <button onClick={() => checkUserRisk(user._id)}>
                    Check Risk
                  </button>
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

## Common Patterns

### Protected Routes with JWT

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" replace />;
  }
  
  // Decode JWT to check role
  const payload = JSON.parse(atob(token.split('.')[1]));
  
  if (requiredRole && payload.role !== requiredRole) {
    return <Navigate to="/unauthorized" replace />;
  }
  
  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/user-dashboard" element={
          <ProtectedRoute requiredRole="user">
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin-dashboard" element={
          <ProtectedRoute requiredRole="admin">
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Task Kanban Board

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  
  useEffect(() => {
    loadTasks();
  }, [userId]);
  
  const loadTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      'in-progress': data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };
  
  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    loadTasks();
  };
  
  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
              <div className="task-actions">
                {status !== 'done' && (
                  <button onClick={() => moveTask(
                    task._id,
                    status === 'todo' ? 'in-progress' : 'done'
                  )}>
                    Move →
                  </button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

// Enable CORS for frontend
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));

app.use(express.json());
```

### ML Model Loading Errors

```python
# ml-service/main.py
import os
from pathlib import Path

@app.on_event("startup")
async def load_models():
    """Load ML models on startup"""
    model_path = Path("models")
    model_path.mkdir(exist_ok=True)
    
    # Check if models exist
    required_models = [
        "ticket_classifier.pkl",
        "tfidf_vectorizer.pkl",
        "risk_detector.pkl"
    ]
    
    for model_file in required_models:
        if not (model_path / model_file).exists():
            print(f"Warning: {model_file} not found. Training default model...")
            # Train or download default model
            train_default_model(model_file)
```

### JWT Token Expiration Handling

```javascript
// utils/api.js
const API_URL = process.env.REACT_APP_API_URL;

export const apiCall = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
      ...options.headers
    }
  });
  
  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }
  
  return response.json();
};
```

### Performance Optimization for Large Datasets

```javascript
// backend/routes/analytics.js
const express = require('express');
const router = express.Router();

// Use aggregation pipeline for better performance
router.get('/dashboard', async (req, res) => {
  try {
    const stats = await Task.aggregate([
      {
        $facet: {
          totalTasks: [{ $count: 'count' }],
          byStatus: [
            { $group: { _id: '$status', count: { $sum: 1 } } }
          ],
          byPriority: [
            { $group: { _id: '$priority', count: { $sum: 1 } } }
          ]
        }
      }
    ]);
    
    res.json(stats[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## Deployment Considerations

### Environment Variables for Production

```bash
# backend/.env.production
PORT=5000
MONGODB_URI=${MONGODB_ATLAS_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=24h
ML_SERVICE_URL=https://your-ml-service.com
NODE_ENV=production
CORS_ORIGIN=https://your-frontend.com

# ml-service/.env.production
PORT=8000
MODEL_PATH=/app/models
MONGODB_URI=${MONGODB_ATLAS_URI}
LOG_LEVEL=warning
```

### Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongo:27017/enterprise_user_management
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - mongo
      - ml-service
  
  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    volumes:
      - ./models:/app/models
  
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000/api
  
  mongo:
    image: mongo:5
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

This system provides a complete enterprise solution with AI-powered insights for modern user and task management needs.
