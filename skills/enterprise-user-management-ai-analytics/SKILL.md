---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user management"
  - "create admin dashboard with analytics"
  - "build task management with AI insights"
  - "configure JWT authentication for user system"
  - "integrate ML service for ticket classification"
  - "deploy user management with AI features"
  - "troubleshoot enterprise user management system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User & Admin Management**: Role-based access control, JWT authentication
- **Task Management**: Kanban boards, time tracking, progress monitoring
- **Support Ticketing**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Real-time Insights**: Organization analytics, audit logs, alerts

**Tech Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

Ensure you have installed:
- Node.js (v14+)
- Python (v3.8+)
- MongoDB (running instance)

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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Key Features & API Endpoints

### Authentication (Backend)

**Login User**
```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
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

**Register User**
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
      role: userData.role || 'user'
    })
  });
  return await response.json();
};
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
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
```

**Delete User**
```javascript
// DELETE /api/users/:id
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

**Create Task**
```javascript
// POST /api/tasks
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
      status: 'todo', // todo, in_progress, done
      priority: taskData.priority, // low, medium, high
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};
```

**Update Task Status**
```javascript
// PUT /api/tasks/:id
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PUT',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return await response.json();
};
```

**Get Ticket AI Classification**
```javascript
// POST /api/tickets/classify (calls ML service)
const classifyTicket = async (ticketText) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/classify', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ text: ticketText })
  });
  return await response.json();
};
```

## ML Service API

### Ticket Classification

```python
# POST /classify-ticket
import requests

def classify_ticket(ticket_text):
    response = requests.post(
        'http://localhost:8000/classify-ticket',
        json={'text': ticket_text}
    )
    return response.json()
    # Returns: { "category": "technical", "priority": "high", "confidence": 0.87 }
```

### Risk Prediction

```python
# POST /predict-risk
def predict_user_risk(user_data):
    response = requests.post(
        'http://localhost:8000/predict-risk',
        json={
            'user_id': user_data['id'],
            'login_frequency': user_data['login_count'],
            'failed_logins': user_data['failed_attempts'],
            'task_completion_rate': user_data['completion_rate'],
            'working_hours': user_data['avg_hours']
        }
    )
    return response.json()
    # Returns: { "risk_score": 0.75, "risk_level": "high", "factors": [...] }
```

### Anomaly Detection

```python
# POST /detect-anomaly
def detect_anomaly(activity_data):
    response = requests.post(
        'http://localhost:8000/detect-anomaly',
        json={
            'timestamp': activity_data['timestamp'],
            'user_id': activity_data['user_id'],
            'action': activity_data['action'],
            'ip_address': activity_data['ip'],
            'location': activity_data['location']
        }
    )
    return response.json()
    # Returns: { "is_anomaly": true, "anomaly_score": 0.92, "reason": "..." }
```

### Burnout Detection

```python
# POST /detect-burnout
def detect_burnout(employee_metrics):
    response = requests.post(
        'http://localhost:8000/detect-burnout',
        json={
            'user_id': employee_metrics['id'],
            'tasks_assigned': employee_metrics['task_count'],
            'tasks_completed': employee_metrics['completed_count'],
            'avg_working_hours': employee_metrics['hours_per_day'],
            'overtime_hours': employee_metrics['overtime'],
            'missed_deadlines': employee_metrics['missed_deadlines']
        }
    )
    return response.json()
    # Returns: { "burnout_risk": "high", "score": 0.88, "recommendations": [...] }
```

### Project Delay Prediction

```python
# POST /predict-delay
def predict_project_delay(project_data):
    response = requests.post(
        'http://localhost:8000/predict-delay',
        json={
            'project_id': project_data['id'],
            'total_tasks': project_data['total_tasks'],
            'completed_tasks': project_data['completed'],
            'days_elapsed': project_data['days_running'],
            'days_remaining': project_data['days_left'],
            'team_size': project_data['team_size'],
            'avg_completion_rate': project_data['completion_rate']
        }
    )
    return response.json()
    # Returns: { "delay_probability": 0.65, "estimated_delay_days": 12, "risk_factors": [...] }
```

## React Component Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchUserProfile();
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    if (data.token) {
      setToken(data.token);
      localStorage.setItem('token', data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Admin Dashboard Component

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const [usersRes, analyticsRes] = await Promise.all([
        fetch('http://localhost:5000/api/users', {
          headers: { 'Authorization': `Bearer ${token}` }
        }),
        fetch('http://localhost:5000/api/analytics/overview', {
          headers: { 'Authorization': `Bearer ${token}` }
        })
      ]);
      
      const usersData = await usersRes.json();
      const analyticsData = await analyticsRes.json();
      
      setUsers(usersData.users);
      setAnalytics(analyticsData);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (!window.confirm('Are you sure you want to delete this user?')) return;
    
    try {
      await fetch(`http://localhost:5000/api/users/${userId}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}` }
      });
      setUsers(users.filter(u => u._id !== userId));
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="analytics-cards">
        <div className="card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers || 0}</p>
        </div>
        <div className="card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks || 0}</p>
        </div>
        <div className="card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets || 0}</p>
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
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
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

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token, user } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await fetch(
        `http://localhost:5000/api/tasks/user/${user._id}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const data = await response.json();
      
      const grouped = {
        todo: data.tasks.filter(t => t.status === 'todo'),
        in_progress: data.tasks.filter(t => t.status === 'in_progress'),
        done: data.tasks.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
        method: 'PUT',
        headers: { 
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      });
      fetchTasks(); // Refresh board
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <div className="task-actions">
            {status !== 'done' && (
              <button onClick={() => moveTask(task._id, 
                status === 'todo' ? 'in_progress' : 'done')}>
                Move →
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in_progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Component

```javascript
// src/components/AIAnalytics.js
import React, { useState, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AIAnalytics = () => {
  const { token, user } = useContext(AuthContext);
  const [analysis, setAnalysis] = useState(null);
  const [loading, setLoading] = useState(false);

  const runBurnoutAnalysis = async () => {
    setLoading(true);
    try {
      const response = await fetch(
        `http://localhost:5000/api/analytics/burnout/${user._id}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const data = await response.json();
      setAnalysis(data);
    } catch (error) {
      console.error('Error running burnout analysis:', error);
    } finally {
      setLoading(false);
    }
  };

  const runRiskAnalysis = async () => {
    setLoading(true);
    try {
      const response = await fetch(
        `http://localhost:5000/api/analytics/risk/${user._id}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const data = await response.json();
      setAnalysis(data);
    } catch (error) {
      console.error('Error running risk analysis:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI Analytics</h2>
      
      <div className="analysis-buttons">
        <button onClick={runBurnoutAnalysis} disabled={loading}>
          Analyze Burnout Risk
        </button>
        <button onClick={runRiskAnalysis} disabled={loading}>
          Analyze Security Risk
        </button>
      </div>

      {loading && <p>Analyzing...</p>}
      
      {analysis && (
        <div className="analysis-results">
          <h3>Analysis Results</h3>
          <pre>{JSON.stringify(analysis, null, 2)}</pre>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=your-secret-key-here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

### ML Service Environment Variables

```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=info
REDIS_URL=redis://localhost:6379
CACHE_ENABLED=true
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Common Patterns

### Protected Route

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// src/services/api.js
const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:5000';

const getAuthHeaders = () => ({
  'Authorization': `Bearer ${localStorage.getItem('token')}`,
  'Content-Type': 'application/json'
});

export const apiService = {
  // Users
  getUsers: async () => {
    const res = await fetch(`${API_BASE}/api/users`, {
      headers: getAuthHeaders()
    });
    return res.json();
  },

  createUser: async (userData) => {
    const res = await fetch(`${API_BASE}/api/users`, {
      method: 'POST',
      headers: getAuthHeaders(),
      body: JSON.stringify(userData)
    });
    return res.json();
  },

  // Tasks
  getTasks: async (userId) => {
    const res = await fetch(`${API_BASE}/api/tasks/user/${userId}`, {
      headers: getAuthHeaders()
    });
    return res.json();
  },

  updateTask: async (taskId, updates) => {
    const res = await fetch(`${API_BASE}/api/tasks/${taskId}`, {
      method: 'PUT',
      headers: getAuthHeaders(),
      body: JSON.stringify(updates)
    });
    return res.json();
  },

  // Tickets
  createTicket: async (ticketData) => {
    const res = await fetch(`${API_BASE}/api/tickets`, {
      method: 'POST',
      headers: getAuthHeaders(),
      body: JSON.stringify(ticketData)
    });
    return res.json();
  }
};
```

## Troubleshooting

### CORS Issues

If you encounter CORS errors, ensure backend CORS is configured:

```javascript
// backend/server.js or app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### JWT Token Expiration

Handle token expiration gracefully:

```javascript
// src/services/api.js
const handleResponse = async (response) => {
  if (response.status === 401) {
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }
  return response.json();
};
```

### MongoDB Connection Issues

Verify MongoDB is running and connection string is correct:

```bash
# Check MongoDB status
sudo systemctl status mongod

# Test connection
mongosh mongodb://localhost:27017/enterprise_users
```

### ML Service Not Responding

Check ML service logs and dependencies:

```bash
cd ml-service
pip install -r requirements.txt --upgrade

# Run with verbose logging
uvicorn main:app --reload --log-level debug
```

### Port Conflicts

If ports are already in use, change them in environment files:

```bash
# Backend
PORT=5001

# ML Service
uvicorn main:app --reload --port 8001

# Frontend
PORT=3001 npm start
```

### Build Errors

Clear cache and reinstall dependencies:

```bash
# Frontend
cd frontend
rm -rf node_modules package-lock.json
npm install

# Backend
cd backend
rm -rf node_modules package-lock.json
npm install
```

### Database Seeding

Create initial admin user:

```javascript
// backend/scripts/seed.js
const mongoose = require('mongoose');
const User = require('./models/User');
const bcrypt = require('bcryptjs');

async function seedAdmin() {
  await mongoose.connect(process.env.MONGODB_URI);
  
  const adminExists = await User.findOne({ email: 'admin@example.com' });
  if (!adminExists) {
    const hashedPassword = await bcrypt.hash('admin123', 10);
    await User.create({
      name: 'Admin User',
      email: 'admin@example.com',
      password: hashedPassword,
      role: 'admin'
    });
    console.log('Admin user created');
  }
  
  process.exit();
}

seedAdmin();
```

Run with: `node backend/scripts/seed.js`
