---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI risk detection to user system"
  - "build admin panel with user management"
  - "integrate AI ticket classification"
  - "configure user management with ML service"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides comprehensive user, task, and support ticket management with integrated AI/ML capabilities. The system includes:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Management**: Kanban boards, time tracking, and progress monitoring
- **Support System**: Ticket creation, classification, and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Organization-wide analytics, audit logs, and monitoring

The architecture consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js) - REST APIs and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML analytics and predictions

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip (for ML service)
python --version
pip --version

# MongoDB (local or cloud)
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup Backend
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# Setup ML Service (in new terminal)
cd ml-service
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload  # Runs on http://localhost:8000

# Setup Frontend (in new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Core API Endpoints

### Authentication APIs

```javascript
// POST /api/auth/register - Register new user
fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'john.doe',
    email: 'john@example.com',
    password: 'securePassword123',
    role: 'user'  // 'user' or 'admin'
  })
});

// POST /api/auth/login - User login
const loginResponse = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'securePassword123'
  })
});
const { token, user } = await loginResponse.json();

// Store token for authenticated requests
localStorage.setItem('authToken', token);
```

### User Management APIs

```javascript
// GET /api/users - Get all users (Admin only)
const getUsers = async () => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
  return await response.json();
};

// GET /api/users/:id - Get user by ID
const getUser = async (userId) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
  return await response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, userData) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify(userData)
  });
  return await response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId) => {
  await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
};
```

### Task Management APIs

```javascript
// POST /api/tasks - Create new task
const createTask = async (taskData) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify({
      title: 'Implement user dashboard',
      description: 'Create responsive user dashboard with analytics',
      assignedTo: 'user_id_here',
      priority: 'high',  // 'low', 'medium', 'high'
      status: 'todo',  // 'todo', 'inprogress', 'done'
      dueDate: '2026-05-01T00:00:00Z'
    })
  });
  return await response.json();
};

// GET /api/tasks - Get tasks (filtered by user)
const getTasks = async (filters = {}) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tasks?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
  return await response.json();
};

// PUT /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};

// POST /api/tasks/:id/time - Track time on task
const trackTime = async (taskId, timeData) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify({
      startTime: '2026-04-15T09:00:00Z',
      endTime: '2026-04-15T11:30:00Z',
      duration: 9000  // seconds
    })
  });
  return await response.json();
};
```

### Support Ticket APIs

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify({
      title: 'Cannot access dashboard',
      description: 'Getting 500 error when trying to load dashboard',
      category: 'technical',  // 'technical', 'access', 'feature'
      priority: 'high'
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (status = 'all') => {
  const response = await fetch(`http://localhost:5000/api/tickets?status=${status}`, {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    }
  });
  return await response.json();
};

// PUT /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('authToken')}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

## AI/ML Service APIs

### Risk Detection

```javascript
// POST /api/ml/risk-detection - Analyze user risk
const analyzeRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: 'user_123',
      loginAttempts: 5,
      failedLogins: 2,
      lastLoginTime: '2026-04-15T08:00:00Z',
      accessPatterns: ['dashboard', 'settings', 'admin'],
      deviceInfo: 'Chrome/Windows'
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection - Detect anomalous behavior
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: 'user_123',
      activityLog: [
        { action: 'login', timestamp: '2026-04-15T02:00:00Z', location: 'US' },
        { action: 'login', timestamp: '2026-04-15T02:15:00Z', location: 'CN' }
      ],
      normalPattern: {
        avgLoginTime: '09:00',
        usualLocations: ['US'],
        typicalActions: ['view_dashboard', 'update_task']
      }
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, reasons: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection - Analyze user burnout risk
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: 'user_123',
      tasksCompleted: 45,
      averageWorkHours: 55,
      overtimeHours: 15,
      weekendWork: true,
      taskDeadlinesMissed: 3,
      stressLevel: 7  // 1-10 scale
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket - Auto-classify support ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: 'Cannot access admin panel',
      description: 'Getting permission denied error when trying to access admin features'
    })
  });
  const result = await response.json();
  // Returns: { category: 'access', priority: 'high', suggestedAssignee: 'admin_id' }
  return result;
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-insights - Get project delay predictions
const getProjectInsights = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      projectId: 'proj_123',
      totalTasks: 50,
      completedTasks: 20,
      averageCompletionTime: 4.5,  // hours per task
      teamSize: 5,
      deadline: '2026-06-01T00:00:00Z'
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.65, estimatedCompletion: '2026-06-10', recommendations: [...] }
  return result;
};
```

## Frontend Components

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('authToken');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets`, config)
      ]);

      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('authToken');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>User Dashboard</h1>
      
      <section className="tasks-section">
        <h2>My Tasks</h2>
        <div className="kanban-board">
          {['todo', 'inprogress', 'done'].map(status => (
            <div key={status} className="kanban-column">
              <h3>{status.toUpperCase()}</h3>
              {tasks
                .filter(task => task.status === status)
                .map(task => (
                  <div key={task._id} className="task-card">
                    <h4>{task.title}</h4>
                    <p>{task.description}</p>
                    <select
                      value={task.status}
                      onChange={(e) => updateTaskStatus(task._id, e.target.value)}
                    >
                      <option value="todo">To Do</option>
                      <option value="inprogress">In Progress</option>
                      <option value="done">Done</option>
                    </select>
                  </div>
                ))}
            </div>
          ))}
        </div>
      </section>

      <section className="tickets-section">
        <h2>My Tickets</h2>
        <div className="tickets-list">
          {tickets.map(ticket => (
            <div key={ticket._id} className="ticket-card">
              <h4>{ticket.title}</h4>
              <p>{ticket.description}</p>
              <span className={`status ${ticket.status}`}>{ticket.status}</span>
            </div>
          ))}
        </div>
      </section>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completedTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('authToken');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
        config
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const analyzeUserRisk = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`,
        { userId }
      );
      alert(`Risk Score: ${response.data.riskScore}, Level: ${response.data.riskLevel}`);
    } catch (error) {
      console.error('Error analyzing risk:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Admin Analytics Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-number">{analytics.activeUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Task Completion</h3>
          <p className="stat-number">
            {analytics.totalTasks > 0 
              ? `${Math.round((analytics.completedTasks / analytics.totalTasks) * 100)}%`
              : '0%'}
          </p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-number">{analytics.openTickets}</p>
        </div>
      </div>

      <section className="risk-monitoring">
        <h2>High Risk Users</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Score</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {analytics.highRiskUsers.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td className="risk-score">{user.riskScore}</td>
                <td>
                  <button onClick={() => analyzeUserRisk(user._id)}>
                    Analyze
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

## Common Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### Axios Instance with Auth

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to every request
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Handle token expiration
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Custom Hooks for Data Fetching

```javascript
// frontend/src/hooks/useTasks.js
import { useState, useEffect } from 'react';
import api from '../utils/api';

export const useTasks = (filters = {}) => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchTasks();
  }, [filters]);

  const fetchTasks = async () => {
    try {
      setLoading(true);
      const queryParams = new URLSearchParams(filters);
      const response = await api.get(`/api/tasks?${queryParams}`);
      setTasks(response.data);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  const createTask = async (taskData) => {
    const response = await api.post('/api/tasks', taskData);
    setTasks([...tasks, response.data]);
    return response.data;
  };

  const updateTask = async (taskId, updates) => {
    const response = await api.put(`/api/tasks/${taskId}`, updates);
    setTasks(tasks.map(t => t._id === taskId ? response.data : t));
    return response.data;
  };

  const deleteTask = async (taskId) => {
    await api.delete(`/api/tasks/${taskId}`);
    setTasks(tasks.filter(t => t._id !== taskId));
  };

  return { tasks, loading, error, createTask, updateTask, deleteTask, refetch: fetchTasks };
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [startTime, setStartTime] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const startTracking = () => {
    setStartTime(Date.now());
    setIsTracking(true);
  };

  const stopTracking = async () => {
    setIsTracking(false);
    const endTime = Date.now();
    const duration = Math.floor((endTime - startTime) / 1000);

    try {
      await api.post(`/api/tasks/${taskId}/time`, {
        startTime: new Date(startTime).toISOString(),
        endTime: new Date(endTime).toISOString(),
        duration
      });
      setElapsedTime(0);
      setStartTime(null);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (ms) => {
    const seconds = Math.floor(ms / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    return `${hours.toString().padStart(2, '0')}:${(minutes % 60).toString().padStart(2, '0')}:${(seconds % 60).toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      {!isTracking ? (
        <button onClick={startTracking}>Start Timer</button>
      ) : (
        <button onClick={stopTracking}>Stop Timer</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Database Schema Examples

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
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
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
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

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number  // in seconds
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  category: {
    type: String,
    enum: ['technical', 'access', 'feature', 'other'],
    default: 'other'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  comments: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    text: String,
    createdAt: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detection.py
from sklearn.ensemble
