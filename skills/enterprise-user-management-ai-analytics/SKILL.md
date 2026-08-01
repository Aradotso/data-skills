---
name: enterprise-user-management-ai-analytics
description: AI-powered user management system with task tracking, ticket management, and predictive analytics for enterprise workflows
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to create tasks and tickets in the enterprise system"
  - "configure JWT authentication for the user management app"
  - "implement AI-based risk detection for users"
  - "set up the kanban board and time tracking features"
  - "deploy the enterprise user management system with ML service"
  - "create admin dashboard with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user administration, task management, and support ticket handling with AI-driven insights. It provides:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Tracking**: Kanban board with time tracking and progress monitoring
- **Ticket System**: Support request management with AI-powered classification
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Centralized monitoring with audit logs and alerts

The system is built with React (frontend), Node.js (backend), MongoDB (database), and FastAPI (ML service).

## Installation

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `.env` file in backend directory:

```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```bash
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
API_KEY=your_ml_service_api_key
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_JWT_SECRET=your_jwt_secret_key_here
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/register - Register new user
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
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

// POST /api/auth/login - User login
const loginUser = async (email, password) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time on task
const logTaskTime = async (taskId, timeSpent, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## AI/ML Service Integration

### AI-Powered Ticket Classification

```javascript
// POST /api/ml/classify-ticket - Auto-classify ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: ticketText.substring(0, 100)
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return result;
};
```

### Risk Detection

```javascript
// POST /api/ml/detect-risk - Analyze user risk score
const detectUserRisk = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginFrequency: userData.loginFrequency,
      failedAttempts: userData.failedAttempts,
      unusualActivity: userData.unusualActivity,
      taskCompletionRate: userData.taskCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout - Analyze user burnout risk
const detectBurnout = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-burnout`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userId,
      workHours: 60, // weekly hours
      taskCount: 15,
      overdueCount: 5,
      avgCompletionTime: 48 // hours
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.82, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly - Detect unusual patterns
const detectAnomaly = async (activityData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      location: activityData.location,
      device: activityData.device,
      activityPattern: activityData.activityPattern
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.9, type: 'unusual_login_time' }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/predict-delay - Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksCompleted: projectData.tasksCompleted,
      tasksRemaining: projectData.tasksRemaining,
      avgVelocity: projectData.avgVelocity,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## React Components - Common Patterns

### Protected Route with JWT

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  // Decode JWT to check role
  const payload = JSON.parse(atob(token.split('.')[1]));
  
  if (requiredRole && payload.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchTasks();
  }, [userId]);
  
  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
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
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };
  
  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, getNextStatus(column))}>
                Move →
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

const getNextStatus = (current) => {
  const flow = { todo: 'in-progress', inProgress: 'done', done: 'done' };
  return flow[current];
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);
  
  const handleStop = async () => {
    setIsRunning(false);
    const minutes = Math.floor(seconds / 60);
    
    // Log time to backend
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: minutes })
    });
    
    setSeconds(0);
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">
        {Math.floor(seconds / 3600)}:{Math.floor((seconds % 3600) / 60)}:{seconds % 60}
      </div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      {isRunning && <button onClick={handleStop}>Stop & Save</button>}
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchAnalytics();
    fetchAlerts();
  }, []);
  
  const fetchAnalytics = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/admin/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAnalytics(data);
  };
  
  const fetchAlerts = async () => {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/get-alerts`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setAlerts(data);
  };
  
  if (!analytics) return <div>Loading...</div>;
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
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
          <h3>Avg Completion Rate</h3>
          <p>{analytics.avgCompletionRate}%</p>
        </div>
      </div>
      
      <div className="alerts-section">
        <h2>AI Alerts</h2>
        {alerts.map(alert => (
          <div key={alert.id} className={`alert alert-${alert.severity}`}>
            <strong>{alert.type}</strong>: {alert.message}
            <span className="alert-time">{new Date(alert.timestamp).toLocaleString()}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Backend Middleware - JWT Authentication

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Access denied. No token provided.' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(400).json({ error: 'Invalid token.' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

## MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  profilePicture: String,
  department: String,
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general'], 
    default: 'general' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  aiClassification: {
    category: String,
    confidence: Number
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration Files

### Backend package.json dependencies

```json
{
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3",
    "dotenv": "^16.0.3",
    "cors": "^2.8.5",
    "axios": "^1.3.4"
  }
}
```

### ML Service requirements.txt

```
fastapi==0.95.0
uvicorn==0.21.1
scikit-learn==1.2.2
river==0.15.0
pandas==2.0.0
numpy==1.24.2
pydantic==1.10.7
python-dotenv==1.0.0
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Handle token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};

// Axios interceptor for auto-refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers.Authorization = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Model Loading

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
import joblib
import os
from pathlib import Path

app = FastAPI()

MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Load models on startup
@app.on_event("startup")
async def load_models():
    global risk_model, burnout_model, ticket_classifier
    
    try:
        risk_model = joblib.load(Path(MODEL_PATH) / 'risk_model.pkl')
        burnout_model = joblib.load(Path(MODEL_PATH) / 'burnout_model.pkl')
        ticket_classifier = joblib.load(Path(MODEL_PATH) / 'ticket_classifier.pkl')
        print("All models loaded successfully")
    except Exception as e:
        print(f"Error loading models: {e}")
        # Use default models or train on startup
        risk_model = train_default_risk_model()
        burnout_model = train_default_burnout_model()
        ticket_classifier = train_default_ticket_classifier()
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### Environment Variables Check

```javascript
// Check required env vars on startup
const requiredEnvVars = ['MONGODB_URI', 'JWT_SECRET', 'ML_SERVICE_URL'];

requiredEnvVars.forEach(varName => {
  if (!process.env[varName]) {
    console.error(`Missing required environment variable: ${varName}`);
    process.exit(1);
  }
});
```

## Production Deployment

### Docker Compose Setup

```yaml
version: '3.8'
services:
  mongodb:
    image: mongo:6.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
  
  backend:
    build: ./backend
    environment:
      MONGODB_URI: mongodb://${MONGO_USER}:${MONGO_PASSWORD}@mongodb:27017/enterprise_user_mgmt
      JWT_SECRET: ${JWT_SECRET}
      ML_SERVICE_URL: http://ml-service:8000
    depends_on:
      - mongodb
  
  ml-service:
    build: ./ml-service
    environment:
      MODEL_PATH: /app/models
      BACKEND_URL: http://backend:5000
  
  frontend:
    build: ./frontend
    environment:
      REACT_APP_API_URL: http://backend:5000/api
      REACT_APP_ML_URL: http://ml-service:8000
    depends_on:
      - backend
      - ml-service

volumes:
  mongo-data:
```

This skill provides comprehensive guidance for using the Enterprise User Management System with AI Analytics, covering authentication, task management, ticket handling, AI integration, and deployment patterns.
