---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics with user management
  - implement JWT authentication for user management
  - create task assignment and tracking system
  - build AI-powered ticket classification
  - set up burnout detection and risk prediction
  - configure MongoDB for user management system
  - implement Kanban board with time tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application combining user/task management with machine learning capabilities. It provides:

- User authentication and role-based access control (RBAC)
- Task management with Kanban boards and time tracking
- Support ticket system with AI classification
- AI-powered analytics: risk detection, anomaly detection, burnout analysis, and predictive project insights
- Admin dashboard with audit logs and security alerts

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or Atlas)

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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/user_management
MODEL_PATH=./models
API_KEY=your_ml_service_api_key
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │  Database   │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │ ML Service  │
                     └─────────────┘
```

## Backend API Usage

### Authentication Endpoints

#### Register User

```javascript
// POST /api/auth/register
const axios = require('axios');

const registerUser = async (userData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    });
    return response.data;
  } catch (error) {
    console.error('Registration error:', error.response.data);
  }
};
```

#### Login

```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email: credentials.email,
      password: credentials.password
    });
    
    // Store JWT token
    const token = response.data.token;
    localStorage.setItem('token', token);
    
    return response.data;
  } catch (error) {
    console.error('Login error:', error.response.data);
  }
};
```

### User Management (Admin)

```javascript
// Authenticated requests require JWT token in header
const api = axios.create({
  baseURL: 'http://localhost:5000/api',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
});

// GET /api/users - Get all users
const getAllUsers = async () => {
  const response = await api.get('/users');
  return response.data;
};

// GET /api/users/:id - Get user by ID
const getUserById = async (userId) => {
  const response = await api.get(`/users/${userId}`);
  return response.data;
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates) => {
  const response = await api.put(`/users/${userId}`, {
    username: updates.username,
    email: updates.email,
    role: updates.role,
    status: updates.status
  });
  return response.data;
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId) => {
  const response = await api.delete(`/users/${userId}`);
  return response.data;
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData) => {
  const response = await api.post('/tasks', {
    title: taskData.title,
    description: taskData.description,
    assignedTo: taskData.userId,
    priority: taskData.priority, // 'low', 'medium', 'high'
    status: 'todo', // 'todo', 'in-progress', 'done'
    dueDate: taskData.dueDate
  });
  return response.data;
};

// GET /api/tasks - Get all tasks (filtered by user if not admin)
const getTasks = async (filters = {}) => {
  const response = await api.get('/tasks', { params: filters });
  return response.data;
};

// PUT /api/tasks/:id - Update task status
const updateTaskStatus = async (taskId, status) => {
  const response = await api.put(`/tasks/${taskId}`, {
    status: status
  });
  return response.data;
};

// POST /api/tasks/:id/time - Track time
const trackTime = async (taskId, timeData) => {
  const response = await api.post(`/tasks/${taskId}/time`, {
    duration: timeData.duration, // in seconds
    date: new Date()
  });
  return response.data;
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData) => {
  const response = await api.post('/tickets', {
    title: ticketData.title,
    description: ticketData.description,
    category: ticketData.category,
    priority: 'medium'
  });
  return response.data;
};

// GET /api/tickets - Get tickets
const getTickets = async (status = null) => {
  const params = status ? { status } : {};
  const response = await api.get('/tickets', { params });
  return response.data;
};

// PUT /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates) => {
  const response = await api.put(`/tickets/${ticketId}`, updates);
  return response.data;
};
```

## ML Service API Usage

### Ticket Classification

```python
# FastAPI endpoint
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    # AI classification logic
    category = classify_text(ticket.title, ticket.description)
    priority = predict_priority(ticket.description)
    
    return {
        "category": category,
        "priority": priority,
        "confidence": 0.85
    }
```

JavaScript client:

```javascript
const classifyTicket = async (ticketText) => {
  const response = await axios.post('http://localhost:8000/classify-ticket', {
    title: ticketText.title,
    description: ticketText.description
  });
  return response.data;
};
```

### Risk Detection

```python
# ML Service endpoint
@app.post("/detect-risk")
async def detect_risk(user_data: dict):
    features = extract_user_features(user_data)
    risk_score = risk_model.predict([features])[0]
    
    return {
        "risk_score": float(risk_score),
        "risk_level": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low",
        "factors": analyze_risk_factors(user_data)
    }
```

JavaScript client:

```javascript
const detectUserRisk = async (userId) => {
  const response = await axios.post('http://localhost:8000/detect-risk', {
    user_id: userId,
    recent_activity: true
  });
  return response.data;
};
```

### Burnout Detection

```python
@app.post("/detect-burnout")
async def detect_burnout(workload_data: dict):
    burnout_score = calculate_burnout_score(
        tasks_count=workload_data['tasks_count'],
        avg_hours=workload_data['avg_hours'],
        missed_deadlines=workload_data['missed_deadlines']
    )
    
    return {
        "burnout_score": burnout_score,
        "status": "critical" if burnout_score > 0.8 else "warning" if burnout_score > 0.6 else "normal",
        "recommendations": generate_recommendations(burnout_score)
    }
```

JavaScript client:

```javascript
const checkBurnout = async (userId) => {
  const response = await axios.post('http://localhost:8000/detect-burnout', {
    user_id: userId,
    time_period: '30d'
  });
  return response.data;
};
```

### Anomaly Detection

```python
@app.post("/detect-anomaly")
async def detect_anomaly(activity_data: dict):
    is_anomalous = anomaly_detector.predict([activity_data['features']])[0]
    
    return {
        "is_anomaly": bool(is_anomalous),
        "anomaly_score": float(anomaly_detector.score_samples([activity_data['features']])[0]),
        "timestamp": activity_data['timestamp']
    }
```

## Frontend React Patterns

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
    checkAuth();
  }, []);

  const checkAuth = async () => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`, {
          headers: { Authorization: `Bearer ${token}` }
        });
        setUser(response.data.user);
      } catch (error) {
        localStorage.removeItem('token');
      }
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
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
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

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
    const token = localStorage.getItem('token');
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`, {
      headers: { Authorization: `Bearer ${token}` },
      params: { userId }
    });

    const grouped = response.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], 'in-progress': [], done: [] });

    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await axios.put(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select
                value={task.status}
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
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

### Time Tracking Component

```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => setIsRunning(true);
  const handlePause = () => setIsRunning(false);

  const handleStop = async () => {
    setIsRunning(false);
    const token = localStorage.getItem('token');
    
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
      { duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    setSeconds(0);
  };

  const formatTime = (secs) => {
    const h = Math.floor(secs / 3600);
    const m = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="controls">
        <button onClick={handleStart} disabled={isRunning}>Start</button>
        <button onClick={handlePause} disabled={!isRunning}>Pause</button>
        <button onClick={handleStop} disabled={!isRunning && seconds === 0}>Stop & Save</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/detect-risk`, { user_id: userId }),
        axios.post(`${process.env.REACT_APP_ML_URL}/detect-burnout`, { user_id: userId })
      ]);

      setAnalytics({
        risk: riskRes.data,
        burnout: burnoutRes.data,
        anomalies: []
      });
    } catch (error) {
      console.error('Analytics fetch error:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk?.risk_level}`}>
          {analytics.risk?.risk_level?.toUpperCase()}
        </div>
        <p>Score: {analytics.risk?.risk_score?.toFixed(2)}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-status ${analytics.burnout?.status}`}>
          {analytics.burnout?.status?.toUpperCase()}
        </div>
        <p>Score: {analytics.burnout?.burnout_score?.toFixed(2)}</p>
        {analytics.burnout?.recommendations && (
          <ul>
            {analytics.burnout.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Schema (MongoDB)

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
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
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
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
  timeTracked: [{
    duration: Number, // seconds
    date: Date
  }],
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, adminOnly };
```

### API Request Helper

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Troubleshooting

### JWT Token Issues

**Problem**: "Token invalid" or "Not authorized" errors

**Solutions**:
```javascript
// Check token expiration
const jwt = require('jsonwebtoken');
const token = localStorage.getItem('token');
const decoded = jwt.decode(token);
console.log('Token expires:', new Date(decoded.exp * 1000));

// Refresh token logic
const refreshToken = async () => {
  const response = await axios.post('/auth/refresh', {
    token: localStorage.getItem('token')
  });
  localStorage.setItem('token', response.data.token);
};
```

### MongoDB Connection Errors

**Problem**: Unable to connect to MongoDB

**Solutions**:
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

### ML Service Connection Issues

**Problem**: Frontend/Backend cannot reach ML service

**Solutions**:
```javascript
// Add timeout and retry logic
const callMLService = async (endpoint, data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}${endpoint}`,
        data,
        { timeout: 10000 }
      );
      return response.data;
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};
```

### CORS Issues

**Problem**: CORS errors when frontend calls backend

**Solution**:
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization

**Problem**: Slow API responses with large datasets

**Solutions**:
```javascript
// Implement pagination
const getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const tasks = await Task.find()
    .skip(skip)
    .limit(limit)
    .populate('assignedTo', 'username email');

  const total = await Task.countDocuments();

  res.json({
    tasks,
    page,
    pages: Math.ceil(total / limit),
    total
  });
};

// Add indexes
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ createdAt: -1 });
```

## Testing

### Backend API Tests

```javascript
// backend/tests/auth.test.js
const request = require('supertest');
const app = require('../server');

describe('Authentication', () => {
  it('should register a new user', async () => {
    const response = await request(app)
      .post('/api/auth/register')
      .send({
        username: 'testuser',
        email: 'test@example.com',
        password: 'password123'
      });

    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty('token');
  });

  it('should login existing user', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123'
      });

    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('token');
  });
});
```

### Frontend Component Tests

```javascript
// frontend/src/components/__tests__/KanbanBoard.test.js
import { render, screen, waitFor } from '@testing-library/react';
import KanbanBoard from '../KanbanBoard';
import axios from 'axios';

jest.mock('axios');

test('renders kanban board with tasks', async () => {
  axios.get.mockResolvedValue({
    data: [
      { _id: '1', title: 'Task 1', status: 'todo' },
      { _id: '2', title: 'Task 2', status: 'in-progress' }
    ]
  });

  render(<KanbanBoard userId="123" />);

  await waitFor(() => {
    expect(screen.getByText('Task 1')).toBeInTheDocument();
    expect(screen.getByText('Task 2')).toBeInTheDocument();
  });
});
```

## Deployment

### Environment Variables Checklist

**Backend**:
- `MONGODB_URI`
- `JWT_SECRET`
- `JWT_EXPIRE`
- `ML_SERVICE_URL`
- `NODE_ENV`

**ML Service**:
- `MONGODB_URI`
- `MODEL_PATH`
- `API_KEY`

**Frontend**:
- `REACT_APP_API_URL`
- `REACT_APP_ML_URL`

### Docker Deployment

```dockerfile
# backend/Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

```dockerfile
# ml-service/Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:5
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/user_management
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/user_management
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost
