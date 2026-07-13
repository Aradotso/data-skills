---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task automation
triggers:
  - "set up enterprise user management with AI analytics"
  - "integrate AI-powered user management system"
  - "configure task management with AI insights"
  - "implement burnout detection and anomaly monitoring"
  - "add AI ticket classification to user management"
  - "create admin dashboard with predictive analytics"
  - "build user management system with ML features"
  - "deploy enterprise task tracking with AI assistant"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project delays.

**Architecture:**
- **Frontend:** React.js application
- **Backend:** Node.js REST API with JWT authentication
- **ML Service:** FastAPI with scikit-learn and River (online learning)
- **Database:** MongoDB
- **Deployment:** Vercel-ready

## Installation

### Prerequisites

```bash
# Required tools
node -v  # v14+ required
npm -v
python -v  # v3.8+ for ML service
mongod --version
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

```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs on http://localhost:3000
```

## Project Structure

```
Enterprise-User-Management-System-with-AI-Analytics/
├── frontend/              # React application
│   ├── src/
│   │   ├── components/   # UI components
│   │   ├── pages/        # Admin/User dashboards
│   │   ├── services/     # API client services
│   │   └── utils/        # Helper functions
│   └── package.json
├── backend/              # Node.js REST API
│   ├── controllers/      # Request handlers
│   ├── models/          # MongoDB schemas
│   ├── routes/          # API routes
│   ├── middleware/      # Auth, validation
│   └── server.js
└── ml-service/          # Python ML service
    ├── main.py          # FastAPI application
    ├── models/          # ML model definitions
    └── requirements.txt
```

## Backend API Reference

### Authentication

```javascript
// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

Response:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "email": "user@example.com",
    "role": "user"
  }
}
```

```javascript
// Register
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass",
  "role": "user"
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: Authorization: Bearer {token}

// Create user
POST /api/users
Headers: Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: Authorization: Bearer {token}

{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
Headers: Authorization: Bearer {token}
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks?userId={userId}
Headers: Authorization: Bearer {token}

// Create task
POST /api/tasks
Headers: Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Implement new feature",
  "description": "Add AI analytics dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
Headers: Authorization: Bearer {token}

{
  "status": "in-progress"  // todo, in-progress, done
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: Authorization: Bearer {token}

{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "medium",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open
Headers: Authorization: Bearer {token}

// Update ticket
PATCH /api/tickets/:ticketId
Headers: Authorization: Bearer {token}

{
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

## Frontend Integration

### API Service Client

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: async (email, password) => {
    const response = await api.post('/auth/login', { email, password });
    localStorage.setItem('token', response.data.token);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
  }
};

export const userService = {
  getAllUsers: () => api.get('/users'),
  createUser: (userData) => api.post('/users', userData),
  updateUser: (userId, data) => api.put(`/users/${userId}`, data),
  deleteUser: (userId) => api.delete(`/users/${userId}`)
};

export const taskService = {
  getTasks: (userId) => api.get(`/tasks?userId=${userId}`),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (taskId, status) => 
    api.patch(`/tasks/${taskId}/status`, { status })
};

export default api;
```

### User Dashboard Component

```javascript
// src/pages/UserDashboard.js
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const userId = localStorage.getItem('userId');
      const response = await taskService.getTasks(userId);
      
      // Organize tasks by status
      const organized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(organized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Tasks</h1>
      <div className="kanban-board">
        <TaskColumn 
          title="To Do" 
          tasks={tasks.todo}
          onStatusChange={handleStatusChange}
        />
        <TaskColumn 
          title="In Progress" 
          tasks={tasks.inProgress}
          onStatusChange={handleStatusChange}
        />
        <TaskColumn 
          title="Done" 
          tasks={tasks.done}
          onStatusChange={handleStatusChange}
        />
      </div>
    </div>
  );
};

const TaskColumn = ({ title, tasks, onStatusChange }) => (
  <div className="task-column">
    <h2>{title}</h2>
    {tasks.map(task => (
      <div key={task.id} className="task-card">
        <h3>{task.title}</h3>
        <p>{task.description}</p>
        <select 
          value={task.status}
          onChange={(e) => onStatusChange(task.id, e.target.value)}
        >
          <option value="todo">To Do</option>
          <option value="in-progress">In Progress</option>
          <option value="done">Done</option>
        </select>
      </div>
    ))}
  </div>
);

export default UserDashboard;
```

## ML Service Integration

### AI Analytics API

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List
import joblib
from river import anomaly, forest

app = FastAPI()

# Load or initialize models
risk_model = joblib.load('./models/risk_predictor.pkl') if os.path.exists('./models/risk_predictor.pkl') else None
anomaly_detector = anomaly.HalfSpaceTrees()

class UserBehavior(BaseModel):
    user_id: str
    login_count: int
    task_completion_rate: float
    avg_task_duration: float
    failed_logins: int

class TaskData(BaseModel):
    task_id: str
    estimated_hours: float
    complexity: str
    assigned_users: int
    progress: float

@app.post("/ai/predict-risk")
async def predict_risk(behavior: UserBehavior):
    """Predict user risk score based on behavior patterns"""
    features = [
        behavior.login_count,
        behavior.task_completion_rate,
        behavior.avg_task_duration,
        behavior.failed_logins
    ]
    
    if risk_model:
        risk_score = risk_model.predict_proba([features])[0][1]
    else:
        # Fallback logic
        risk_score = (behavior.failed_logins * 0.3 + 
                     (1 - behavior.task_completion_rate) * 0.5)
    
    return {
        "user_id": behavior.user_id,
        "risk_score": float(risk_score),
        "risk_level": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
    }

@app.post("/ai/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    features = {
        'login_count': behavior.login_count,
        'task_completion_rate': behavior.task_completion_rate,
        'failed_logins': behavior.failed_logins
    }
    
    score = anomaly_detector.score_one(features)
    anomaly_detector.learn_one(features)
    
    return {
        "user_id": behavior.user_id,
        "anomaly_score": float(score),
        "is_anomaly": score > 0.5
    }

@app.post("/ai/predict-burnout")
async def predict_burnout(user_id: str, tasks: List[TaskData]):
    """Analyze workload and predict burnout risk"""
    total_hours = sum(t.estimated_hours for t in tasks)
    avg_progress = sum(t.progress for t in tasks) / len(tasks) if tasks else 0
    
    burnout_score = min(1.0, total_hours / 160)  # 160 hours/month baseline
    if avg_progress < 0.5:
        burnout_score += 0.2
    
    return {
        "user_id": user_id,
        "burnout_score": float(burnout_score),
        "workload_hours": total_hours,
        "recommendation": "reduce workload" if burnout_score > 0.7 else "maintain pace"
    }

@app.post("/ai/classify-ticket")
async def classify_ticket(title: str, description: str):
    """Classify support ticket and suggest routing"""
    text = f"{title} {description}".lower()
    
    # Simple keyword-based classification
    if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
        category = 'technical'
        priority = 'high'
    elif any(word in text for word in ['access', 'login', 'password', 'permission']):
        category = 'access'
        priority = 'medium'
    elif any(word in text for word in ['feature', 'request', 'enhancement']):
        category = 'feature_request'
        priority = 'low'
    else:
        category = 'general'
        priority = 'medium'
    
    return {
        "category": category,
        "priority": priority,
        "suggested_department": "IT" if category in ['technical', 'access'] else "Product"
    }
```

### Calling ML Service from Backend

```javascript
// backend/controllers/aiController.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

exports.analyzeUserRisk = async (req, res) => {
  try {
    const { userId } = req.params;
    
    // Fetch user behavior data
    const behavior = await getUserBehaviorMetrics(userId);
    
    // Call ML service
    const mlResponse = await axios.post(`${ML_SERVICE_URL}/ai/predict-risk`, {
      user_id: userId,
      login_count: behavior.loginCount,
      task_completion_rate: behavior.completionRate,
      avg_task_duration: behavior.avgDuration,
      failed_logins: behavior.failedLogins
    });
    
    res.json(mlResponse.data);
  } catch (error) {
    console.error('AI risk analysis failed:', error);
    res.status(500).json({ error: 'Failed to analyze user risk' });
  }
};

exports.classifyTicket = async (req, res) => {
  try {
    const { title, description } = req.body;
    
    const mlResponse = await axios.post(`${ML_SERVICE_URL}/ai/classify-ticket`, {
      title,
      description
    });
    
    // Auto-assign ticket based on classification
    const ticket = await Ticket.create({
      ...req.body,
      category: mlResponse.data.category,
      priority: mlResponse.data.priority,
      assignedDepartment: mlResponse.data.suggested_department
    });
    
    res.status(201).json({
      ticket,
      aiClassification: mlResponse.data
    });
  } catch (error) {
    console.error('Ticket classification failed:', error);
    res.status(500).json({ error: 'Failed to classify ticket' });
  }
};

async function getUserBehaviorMetrics(userId) {
  // Implementation to aggregate user behavior data
  const user = await User.findById(userId);
  const tasks = await Task.find({ assignedTo: userId });
  const loginLogs = await LoginLog.find({ userId });
  
  return {
    loginCount: loginLogs.filter(l => l.success).length,
    failedLogins: loginLogs.filter(l => !l.success).length,
    completionRate: tasks.filter(t => t.status === 'done').length / tasks.length,
    avgDuration: tasks.reduce((sum, t) => sum + t.duration, 0) / tasks.length
  };
}
```

### Frontend AI Integration

```javascript
// src/services/aiService.js
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;

export const aiService = {
  analyzeUserRisk: async (userId) => {
    const response = await axios.get(`/api/ai/risk/${userId}`);
    return response.data;
  },
  
  detectAnomalies: async (userId) => {
    const response = await axios.get(`/api/ai/anomaly/${userId}`);
    return response.data;
  },
  
  getBurnoutAnalysis: async (userId) => {
    const response = await axios.get(`/api/ai/burnout/${userId}`);
    return response.data;
  }
};

// Usage in Admin Dashboard
const AdminDashboard = () => {
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  
  useEffect(() => {
    loadAIInsights();
  }, []);
  
  const loadAIInsights = async () => {
    const users = await userService.getAllUsers();
    const analyses = await Promise.all(
      users.data.map(user => aiService.analyzeUserRisk(user.id))
    );
    setRiskAnalysis(analyses);
  };
  
  return (
    <div className="admin-dashboard">
      <h1>AI Analytics Dashboard</h1>
      <div className="risk-alerts">
        {riskAnalysis.filter(a => a.risk_level === 'high').map(alert => (
          <div key={alert.user_id} className="alert alert-danger">
            High risk user detected: {alert.user_id}
            <span>Risk Score: {(alert.risk_score * 100).toFixed(0)}%</span>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

// Usage in App.js
<Routes>
  <Route path="/login" element={<Login />} />
  <Route path="/user/dashboard" element={
    <ProtectedRoute requiredRole="user">
      <UserDashboard />
    </ProtectedRoute>
  } />
  <Route path="/admin/dashboard" element={
    <ProtectedRoute requiredRole="admin">
      <AdminDashboard />
    </ProtectedRoute>
  } />
</Routes>
```

### Real-time Notifications

```javascript
// backend/utils/notificationService.js
const EventEmitter = require('events');
const notificationEmitter = new EventEmitter();

exports.sendNotification = (userId, message, type) => {
  notificationEmitter.emit('notification', {
    userId,
    message,
    type,
    timestamp: new Date()
  });
};

// WebSocket setup for real-time updates
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  socket.on('subscribe', (userId) => {
    socket.join(`user-${userId}`);
  });
});

notificationEmitter.on('notification', (notification) => {
  io.to(`user-${notification.userId}`).emit('notification', notification);
});
```

## Deployment

### Vercel Deployment (Frontend)

```json
// vercel.json
{
  "version": 2,
  "builds": [
    {
      "src": "frontend/package.json",
      "use": "@vercel/static-build",
      "config": { "distDir": "build" }
    }
  ],
  "env": {
    "REACT_APP_API_URL": "@api_url",
    "REACT_APP_ML_URL": "@ml_url"
  }
}
```

```bash
cd frontend
vercel --prod
```

### Backend Deployment

```bash
# Using Heroku
cd backend
heroku create your-app-name
heroku config:set MONGODB_URI=your_mongo_uri
heroku config:set JWT_SECRET=your_secret
git push heroku main
```

### ML Service Deployment

```bash
# Using Railway or Render
# Ensure requirements.txt includes:
# fastapi
# uvicorn
# scikit-learn
# river
# joblib
# python-dotenv

# Procfile
web: uvicorn main:app --host 0.0.0.0 --port $PORT
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Add token refresh logic
api.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

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
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Performance

```python
# Cache model predictions
from functools import lru_cache

@lru_cache(maxsize=100)
def get_cached_risk_prediction(user_id: str, behavior_hash: str):
    # Implementation
    pass

# Use batch processing for multiple users
@app.post("/ai/batch-risk-analysis")
async def batch_risk_analysis(users: List[UserBehavior]):
    results = []
    for user in users:
        result = await predict_risk(user)
        results.append(result)
    return results
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

## Testing

### Backend API Tests

```javascript
// backend/tests/user.test.js
const request = require('supertest');
const app = require('../server');

describe('User API', () => {
  let authToken;
  
  beforeAll(async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'admin@test.com', password: 'admin123' });
    authToken = response.body.token;
  });
  
  test('GET /api/users - should return all users', async () => {
    const response = await request(app)
      .get('/api/users')
      .set('Authorization', `Bearer ${authToken}`);
    
    expect(response.status).toBe(200);
    expect(Array.isArray(response.body)).toBe(true);
  });
});
```

### Frontend Component Tests

```javascript
// frontend/src/tests/UserDashboard.test.js
import { render, screen, waitFor } from '@testing-library/react';
import UserDashboard from '../pages/UserDashboard';
import { taskService } from '../services/api';

jest.mock('../services/api');

test('renders user tasks', async () => {
  taskService.getTasks.mockResolvedValue({
    data: [
      { id: '1', title: 'Test Task', status: 'todo' }
    ]
  });
  
  render(<UserDashboard />);
  
  await waitFor(() => {
    expect(screen.getByText('Test Task')).toBeInTheDocument();
  });
});
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, integrating, and troubleshooting all aspects of the system.
