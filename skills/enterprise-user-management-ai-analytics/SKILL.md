---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create admin dashboard with user analytics"
  - "implement AI-based ticket classification"
  - "build user management system with risk detection"
  - "add burnout detection to user system"
  - "configure AI insights for task management"
  - "deploy user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: JWT-based authentication, role-based access control (RBAC), user CRUD operations
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support System**: Smart ticket classification, automated routing, priority detection
- **AI Analytics**: Risk scoring, anomaly detection, burnout prediction, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, security alerts

The system uses a microservices architecture with React frontend, Node.js backend, and FastAPI ML service.

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

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup (Python/FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Backend REST API (Node.js)

#### Authentication

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

#### User Management (Admin)

```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user

// Example: Update user role
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    role: 'admin',
    status: 'active'
  })
});
```

#### Task Management

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement feature X",
  "description": "Add new dashboard widget",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PATCH /api/tasks/:id/status - Update task status
{
  "status": "in-progress"
}

// GET /api/tasks/user/:userId - Get user tasks
```

#### Support Tickets

```javascript
// POST /api/tickets - Create ticket
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "category": "technical",
  "priority": "high"
}

// GET /api/tickets - List all tickets (admin)
// GET /api/tickets/user/:userId - User's tickets
// PATCH /api/tickets/:id - Update ticket
```

### ML Service API (FastAPI)

```python
# POST /api/ml/classify-ticket - AI ticket classification
{
  "title": "Cannot login to system",
  "description": "Getting authentication error when trying to access dashboard"
}
# Returns: { "category": "technical", "priority": "high", "confidence": 0.92 }

# POST /api/ml/risk-score - User risk prediction
{
  "userId": "user_123",
  "failedLogins": 3,
  "unusualActivity": true,
  "lastLogin": "2026-04-15T10:30:00Z"
}
# Returns: { "riskScore": 0.78, "riskLevel": "high", "factors": [...] }

# POST /api/ml/burnout-detection - Workload analysis
{
  "userId": "user_123",
  "tasksCompleted": 45,
  "avgWorkHours": 52,
  "missedDeadlines": 5
}
# Returns: { "burnoutRisk": 0.85, "recommendation": "reduce workload" }

# POST /api/ml/anomaly-detection - Behavior anomalies
{
  "userId": "user_123",
  "loginTime": "03:00",
  "location": "unusual_ip",
  "accessPattern": "abnormal"
}
# Returns: { "isAnomaly": true, "anomalyScore": 0.91 }
```

## Configuration

### Backend Configuration (backend/config/db.js)

```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('Database connection failed:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware (backend/middleware/auth.js)

```javascript
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

### ML Service Configuration (ml-service/main.py)

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from river import anomaly, compose, preprocessing

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load pre-trained models
try:
    ticket_classifier = joblib.load('models/ticket_classifier.pkl')
    risk_model = joblib.load('models/risk_model.pkl')
except FileNotFoundError:
    print("Models not found, using default configurations")

# Online learning model for anomaly detection
anomaly_detector = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees(seed=42)
)

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskRequest(BaseModel):
    userId: str
    failedLogins: int
    unusualActivity: bool
    lastLogin: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    try:
        # Feature extraction
        text = f"{ticket.title} {ticket.description}"
        # Predict category and priority
        category = "technical"  # Use actual model prediction
        priority = "high"
        confidence = 0.92
        
        return {
            "category": category,
            "priority": priority,
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-score")
async def calculate_risk(data: RiskRequest):
    try:
        # Calculate risk score
        risk_score = (data.failedLogins * 0.3 + 
                     (0.5 if data.unusualActivity else 0))
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": ["failed_logins", "unusual_activity"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration Patterns

### React API Service (frontend/src/services/api.js)

```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: async (email, password) => {
    const response = await api.post('/api/auth/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

export const userService = {
  getUsers: () => api.get('/api/users'),
  getUser: (id) => api.get(`/api/users/${id}`),
  updateUser: (id, data) => api.put(`/api/users/${id}`, data),
  deleteUser: (id) => api.delete(`/api/users/${id}`)
};

export const taskService = {
  createTask: (data) => api.post('/api/tasks', data),
  getUserTasks: (userId) => api.get(`/api/tasks/user/${userId}`),
  updateTaskStatus: (id, status) => 
    api.patch(`/api/tasks/${id}/status`, { status })
};

export const mlService = {
  classifyTicket: async (title, description) => {
    const response = await axios.post(`${ML_URL}/api/ml/classify-ticket`, {
      title,
      description
    });
    return response.data;
  },
  
  getRiskScore: async (userData) => {
    const response = await axios.post(`${ML_URL}/api/ml/risk-score`, userData);
    return response.data;
  }
};
```

### Kanban Board Component (frontend/src/components/KanbanBoard.jsx)

```javascript
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    try {
      const response = await taskService.getUserTasks(userId);
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const renderColumn = (status, title, taskList) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => handleDrop(e, status)}
    >
      <h3>{title}</h3>
      {taskList.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority ${task.priority}`}>
            {task.priority}
          </span>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('in-progress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard Component (frontend/src/components/AdminDashboard.jsx)

```javascript
import React, { useState, useEffect } from 'react';
import { userService, mlService } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const usersResponse = await userService.getUsers();
      setUsers(usersResponse.data);
      
      // Calculate risk scores for users
      const risks = await Promise.all(
        usersResponse.data.map(async (user) => {
          const riskData = await mlService.getRiskScore({
            userId: user._id,
            failedLogins: user.failedLogins || 0,
            unusualActivity: user.unusualActivity || false,
            lastLogin: user.lastLogin
          });
          return { ...user, risk: riskData };
        })
      );
      
      setRiskUsers(risks.filter(u => u.risk.riskLevel === 'high'));
    } catch (error) {
      console.error('Failed to load dashboard data:', error);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await userService.deleteUser(userId);
        loadDashboardData();
      } catch (error) {
        console.error('Failed to delete user:', error);
      }
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{users.length}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-number">{riskUsers.length}</p>
        </div>
      </div>

      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Risk Level</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status}</td>
                <td className={user.risk?.riskLevel}>
                  {user.risk?.riskLevel || 'low'}
                </td>
                <td>
                  <button onClick={() => handleDeleteUser(user._id)}>
                    Delete
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

## Database Models

### User Model (backend/models/User.js)

```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model (backend/models/Task.js)

```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
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
  timeTracked: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
mongod --version

# If using MongoDB Atlas, verify connection string
echo $MONGODB_URI

# Test connection
mongosh $MONGODB_URI
```

### JWT Token Errors

```javascript
// Verify token format in requests
const token = localStorage.getItem('token');
console.log('Token:', token ? 'Present' : 'Missing');

// Check token expiration
import jwt_decode from 'jwt-decode';
const decoded = jwt_decode(token);
console.log('Token expires:', new Date(decoded.exp * 1000));
```

### ML Service Not Responding

```bash
# Check ML service is running
curl http://localhost:8000/docs

# Verify Python dependencies
pip list | grep -E "fastapi|scikit-learn|river"

# Check logs
tail -f ml-service/logs/app.log
```

### CORS Issues

```javascript
// Backend: Ensure CORS is configured
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));

// Frontend: Check API URL configuration
console.log('API URL:', process.env.REACT_APP_API_URL);
```

### Model Training Issues

```python
# Train ticket classifier
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib

# Sample training data
tickets = [
    ("Cannot login", "technical"),
    ("Password reset needed", "account"),
    ("Feature request", "enhancement")
]

texts, labels = zip(*tickets)
vectorizer = TfidfVectorizer()
X = vectorizer.fit_transform(texts)

model = MultinomialNB()
model.fit(X, labels)

joblib.dump((vectorizer, model), 'models/ticket_classifier.pkl')
```

### Performance Optimization

```javascript
// Backend: Add database indexing
userSchema.index({ email: 1 });
taskSchema.index({ assignedTo: 1, status: 1 });

// Frontend: Implement pagination
const [page, setPage] = useState(1);
const limit = 20;

const loadUsers = async () => {
  const response = await api.get(`/api/users?page=${page}&limit=${limit}`);
  setUsers(response.data);
};
```
