---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - implement AI analytics for user risk detection
  - create a user management dashboard with AI insights
  - integrate ticket classification and anomaly detection
  - build a kanban board with time tracking
  - add burnout analysis to user management
  - configure JWT authentication for enterprise users
  - set up AI-powered task management system
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-powered insights. It provides:

- **User Management**: Role-based access control, authentication, and user operations
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights

**Stack**: React frontend, Node.js backend, MongoDB database, FastAPI ML service

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance (local or cloud)

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
MONGO_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
DB_URI=${MONGODB_URI}
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
```

## Key Architecture Patterns

### Authentication Flow

**Backend JWT Implementation** (`backend/middleware/auth.js`):

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

**Login Route** (`backend/routes/auth.js`):

```javascript
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        email: user.email,
        name: user.name,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### User Management API

**User Model** (`backend/models/User.js`):

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  tasksAssigned: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Task' }],
  performanceScore: { type: Number, default: 0 },
  riskLevel: { type: String, enum: ['low', 'medium', 'high'], default: 'low' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

module.exports = mongoose.model('User', userSchema);
```

**User CRUD Operations** (`backend/routes/users.js`):

```javascript
const express = require('express');
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

const router = express.Router();

// Get all users (Admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find()
      .select('-password')
      .populate('tasksAssigned');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
router.get('/:id', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.params.id)
      .select('-password')
      .populate('tasksAssigned');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, department, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, department, status },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management with Kanban

**Task Model** (`backend/models/Task.js`):

```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

**Task Routes** (`backend/routes/tasks.js`):

```javascript
const express = require('express');
const Task = require('../models/Task');
const User = require('../models/User');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    
    // Add to user's assigned tasks
    if (task.assignedTo) {
      await User.findByIdAndUpdate(task.assignedTo, {
        $push: { tasksAssigned: task._id }
      });
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get tasks (filtered by user role)
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const update = { status };
    if (status === 'done') {
      update.completedAt = new Date();
    }
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      update,
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.patch('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { minutes } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeTracked: minutes } },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

**Ticket Model** (`backend/models/Ticket.js`):

```javascript
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'urgent'], 
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
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: String,
  sentiment: String,
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### AI Integration Patterns

**AI Service Client** (`backend/services/aiService.js`):

```javascript
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

class AIService {
  // Classify ticket using AI
  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/classify-ticket`, {
        title: ticketData.title,
        description: ticketData.description
      });
      return response.data;
    } catch (error) {
      console.error('AI classification failed:', error.message);
      return { category: 'general', confidence: 0 };
    }
  }
  
  // Detect user risk
  async detectRisk(userId, userMetrics) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/detect-risk`, {
        user_id: userId,
        metrics: userMetrics
      });
      return response.data;
    } catch (error) {
      console.error('Risk detection failed:', error.message);
      return { risk_level: 'low', score: 0 };
    }
  }
  
  // Analyze burnout
  async analyzeBurnout(userId, workloadData) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/analyze-burnout`, {
        user_id: userId,
        tasks_count: workloadData.tasksCount,
        hours_worked: workloadData.hoursWorked,
        completion_rate: workloadData.completionRate
      });
      return response.data;
    } catch (error) {
      console.error('Burnout analysis failed:', error.message);
      return { burnout_risk: 'low', recommendations: [] };
    }
  }
  
  // Detect anomalies
  async detectAnomalies(activityData) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/detect-anomaly`, {
        activities: activityData
      });
      return response.data;
    } catch (error) {
      console.error('Anomaly detection failed:', error.message);
      return { is_anomaly: false, score: 0 };
    }
  }
}

module.exports = new AIService();
```

**Ticket Creation with AI** (`backend/routes/tickets.js`):

```javascript
const express = require('express');
const Ticket = require('../models/Ticket');
const aiService = require('../services/aiService');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Get AI classification
    const aiResult = await aiService.classifyTicket({ title, description });
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      category: aiResult.category || 'general',
      aiClassification: aiResult.category,
      priority: aiResult.priority || 'medium'
    });
    
    await ticket.save();
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Frontend Integration

**API Client** (`frontend/src/services/api.js`):

```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

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

// Auth endpoints
export const login = (email, password) => 
  api.post('/auth/login', { email, password });

export const register = (userData) => 
  api.post('/auth/register', userData);

// User endpoints
export const getUsers = () => api.get('/users');
export const getUser = (id) => api.get(`/users/${id}`);
export const updateUser = (id, data) => api.put(`/users/${id}`, data);
export const deleteUser = (id) => api.delete(`/users/${id}`);

// Task endpoints
export const getTasks = () => api.get('/tasks');
export const createTask = (taskData) => api.post('/tasks', taskData);
export const updateTaskStatus = (id, status) => 
  api.patch(`/tasks/${id}/status`, { status });
export const trackTime = (id, minutes) => 
  api.patch(`/tasks/${id}/time`, { minutes });

// Ticket endpoints
export const getTickets = () => api.get('/tickets');
export const createTicket = (ticketData) => api.post('/tickets', ticketData);
export const updateTicket = (id, data) => api.put(`/tickets/${id}`, data);

// Analytics endpoints
export const getUserAnalytics = (userId) => api.get(`/analytics/user/${userId}`);
export const getOrgAnalytics = () => api.get('/analytics/organization');

export default api;
```

**React Dashboard Component** (`frontend/src/components/Dashboard.jsx`):

```javascript
import React, { useState, useEffect } from 'react';
import { getTasks, getUserAnalytics } from '../services/api';

const Dashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const [tasksRes, analyticsRes] = await Promise.all([
        getTasks(),
        getUserAnalytics('me')
      ]);

      // Group tasks by status
      const grouped = tasksRes.data.reduce((acc, task) => {
        const status = task.status === 'in-progress' ? 'inProgress' : task.status;
        acc[status].push(task);
        return acc;
      }, { todo: [], inProgress: [], done: [] });

      setTasks(grouped);
      setAnalytics(analyticsRes.data);
    } catch (error) {
      console.error('Failed to load dashboard:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <div className="analytics-summary">
        <div className="metric-card">
          <h3>Tasks Completed</h3>
          <p>{analytics?.tasksCompleted || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Hours Tracked</h3>
          <p>{Math.round((analytics?.timeTracked || 0) / 60)}h</p>
        </div>
        <div className="metric-card">
          <h3>Performance Score</h3>
          <p>{analytics?.performanceScore || 0}%</p>
        </div>
      </div>

      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
        <div className="kanban-column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
        <div className="kanban-column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} onUpdate={loadDashboardData} />
          ))}
        </div>
      </div>
    </div>
  );
};

export default Dashboard;
```

**Task Card Component** (`frontend/src/components/TaskCard.jsx`):

```javascript
import React, { useState } from 'react';
import { updateTaskStatus, trackTime } from '../services/api';

const TaskCard = ({ task, onUpdate }) => {
  const [tracking, setTracking] = useState(false);
  const [startTime, setStartTime] = useState(null);

  const handleStatusChange = async (newStatus) => {
    try {
      await updateTaskStatus(task._id, newStatus);
      onUpdate();
    } catch (error) {
      console.error('Failed to update status:', error);
    }
  };

  const toggleTracking = async () => {
    if (tracking) {
      // Stop tracking and save time
      const elapsed = Math.floor((Date.now() - startTime) / 60000); // minutes
      try {
        await trackTime(task._id, elapsed);
        onUpdate();
      } catch (error) {
        console.error('Failed to track time:', error);
      }
    } else {
      setStartTime(Date.now());
    }
    setTracking(!tracking);
  };

  return (
    <div className={`task-card priority-${task.priority}`}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className="priority">{task.priority}</span>
        <span className="time">{Math.round(task.timeTracked / 60)}h tracked</span>
      </div>
      <div className="task-actions">
        <button onClick={toggleTracking}>
          {tracking ? 'Stop' : 'Start'} Timer
        </button>
        {task.status !== 'done' && (
          <button onClick={() => handleStatusChange(
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            {task.status === 'todo' ? 'Start' : 'Complete'}
          </button>
        )}
      </div>
    </div>
  );
};

export default TaskCard;
```

## Configuration

### Environment Variables

**Backend** (`.env`):
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-mgmt
JWT_SECRET=your-secret-key-here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend** (`.env`):
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service** (`.env`):
```env
MODEL_PATH=./models
DB_URI=mongodb://localhost:27017/enterprise-mgmt
PYTHONUNBUFFERED=1
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/App.js
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute><Dashboard /></ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly><AdminPanel /></ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Updates

```javascript
// backend/routes/tasks.js - emit updates via WebSocket or polling
const notifyTaskUpdate = (taskId, update) => {
  // If using Socket.io
  io.emit('task-updated', { taskId, update });
};

// frontend - subscribe to updates
import { useEffect } from 'react';
import io from 'socket.io-client';

const useRealtimeUpdates = (onUpdate) => {
  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL);
    
    socket.on('task-updated', (data) => {
      onUpdate(data);
    });
    
    return () => socket.disconnect();
  }, [onUpdate]);
};
```

## Troubleshooting

### Common Issues

**JWT Token Expiration**:
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

**MongoDB Connection Issues**:
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
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

**AI Service Timeout**:
```javascript
// Increase timeout for ML requests
const aiServiceWithTimeout = axios.create({
  baseURL: ML_SERVICE_URL,
  timeout: 30000 // 30 seconds
});
```

**CORS Issues**:
```javascript
// backend/server.js
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization

**Pagination for Large Datasets**:
```javascript
router.get('/users', authMiddleware, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;
  
  const [users, total] = await Promise.all([
    User.find().skip(skip).limit(limit).select('-password'),
    User.countDocuments()
  ]);
  
  res.json({
    users,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
});
```

**Caching Analytics**:
```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 300 }); // 5 min cache

router.get('/analytics/organization', authMiddleware, async (req, res) => {
  const cacheKey = 'org-analytics';
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  const analytics = await calculateOrgAnalytics();
  cache.set(cacheKey, analytics);
  res.json(analytics);
});
```

This skill provides comprehensive guidance for working with the Enterprise User Management System, covering authentication, CRUD operations, AI integration, and frontend development patterns.
