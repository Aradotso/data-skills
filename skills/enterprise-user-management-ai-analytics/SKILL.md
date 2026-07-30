---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management with AI"
  - "create user management dashboard with analytics"
  - "implement AI-powered ticket classification"
  - "build task tracking system with burnout detection"
  - "add AI insights to user management"
  - "configure user management with ML service"
  - "integrate AI analytics for user behavior"
  - "deploy enterprise user management system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## What It Does

This system provides:
- **User & Admin Dashboards**: Role-based access with JWT authentication
- **Task Management**: Kanban boards, time tracking, and progress monitoring
- **Support Tickets**: AI-powered classification and intelligent routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Real-time Insights**: Performance metrics, audit logs, and security alerts

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+ with pip
- MongoDB instance

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOL
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOL

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOL
MODEL_PATH=./models
LOG_LEVEL=INFO
EOL

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOL
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOL

# Start frontend
npm start
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
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

### User Management (Admin)

```javascript
// GET /api/users - List all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
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
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
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
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};

// POST /api/tasks/:id/time - Track time
const trackTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      duration: timeData.duration, // seconds
      startTime: timeData.startTime,
      endTime: timeData.endTime
    })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service API

### Ticket Classification

```javascript
// POST /classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      metadata: {
        priority: 'high',
        timestamp: new Date().toISOString()
      }
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', confidence: 0.85, suggested_assignee: 'team_id' }
  return data;
};
```

### Risk Detection

```javascript
// POST /predict-risk
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userData.userId,
      login_frequency: userData.loginFrequency,
      task_completion_rate: userData.taskCompletionRate,
      failed_logins: userData.failedLogins,
      unusual_hours: userData.unusualHours,
      access_patterns: userData.accessPatterns
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.65, risk_level: 'medium', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      features: {
        login_time: userActivity.loginTime,
        ip_address: userActivity.ipAddress,
        location: userActivity.location,
        actions_per_hour: userActivity.actionsPerHour,
        data_volume: userActivity.dataVolume
      }
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, alerts: [...] }
  return data;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout
const detectBurnout = async (userMetrics) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userMetrics.userId,
      metrics: {
        tasks_completed: userMetrics.tasksCompleted,
        hours_worked: userMetrics.hoursWorked,
        overtime_hours: userMetrics.overtimeHours,
        deadline_missed: userMetrics.deadlineMissed,
        task_load: userMetrics.taskLoad,
        days_since_break: userMetrics.daysSinceBreak
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Project Delay Prediction

```javascript
// POST /predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      features: {
        total_tasks: projectData.totalTasks,
        completed_tasks: projectData.completedTasks,
        team_size: projectData.teamSize,
        days_remaining: projectData.daysRemaining,
        avg_task_completion_time: projectData.avgTaskCompletionTime,
        blockers: projectData.blockers
      }
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.72, estimated_delay_days: 5, risk_factors: [...] }
  return data;
};
```

## React Component Examples

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      // Fetch tasks
      const tasksRes = await fetch('http://localhost:5000/api/tasks/my-tasks', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const tasksData = await tasksRes.json();
      setTasks(tasksData);

      // Fetch tickets
      const ticketsRes = await fetch('http://localhost:5000/api/tickets/my-tickets', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const ticketsData = await ticketsRes.json();
      setTickets(ticketsData);

      // Fetch AI analytics
      const analyticsRes = await fetch('http://localhost:8000/user-analytics', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: getUserId() })
      });
      const analyticsData = await analyticsRes.json();
      setAnalytics(analyticsData);
    } catch (error) {
      console.error('Error fetching user data:', error);
    }
  };

  const getUserId = () => {
    const tokenPayload = JSON.parse(atob(token.split('.')[1]));
    return tokenPayload.userId;
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {analytics && analytics.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected. Consider taking a break.
        </div>
      )}

      <div className="tasks-section">
        <h2>My Tasks</h2>
        <div className="kanban-board">
          <TaskColumn title="To Do" tasks={tasks.filter(t => t.status === 'todo')} />
          <TaskColumn title="In Progress" tasks={tasks.filter(t => t.status === 'in-progress')} />
          <TaskColumn title="Done" tasks={tasks.filter(t => t.status === 'done')} />
        </div>
      </div>

      <div className="tickets-section">
        <h2>My Tickets</h2>
        <TicketList tickets={tickets} />
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch all users
      const usersRes = await fetch('http://localhost:5000/api/users', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const usersData = await usersRes.json();
      setUsers(usersData);

      // Analyze each user for risks
      const risks = await Promise.all(
        usersData.map(async (user) => {
          const riskRes = await fetch('http://localhost:8000/predict-risk', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              user_id: user._id,
              login_frequency: user.loginFrequency || 0,
              task_completion_rate: user.taskCompletionRate || 0,
              failed_logins: user.failedLogins || 0
            })
          });
          const riskData = await riskRes.json();
          return { user, risk: riskData };
        })
      );

      // Filter high-risk users
      const highRisk = risks.filter(r => r.risk.risk_level === 'high');
      setRiskAlerts(highRisk);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Organization Analytics</h1>

      <div className="risk-alerts">
        <h2>High-Risk Users</h2>
        {riskAlerts.length > 0 ? (
          <ul>
            {riskAlerts.map(alert => (
              <li key={alert.user._id}>
                <strong>{alert.user.username}</strong> - 
                Risk Score: {(alert.risk.risk_score * 100).toFixed(0)}%
                <br />
                Factors: {alert.risk.factors.join(', ')}
              </li>
            ))}
          </ul>
        ) : (
          <p>No high-risk users detected</p>
        )}
      </div>

      <div className="user-stats">
        <h2>User Statistics</h2>
        <p>Total Users: {users.length}</p>
        <p>Active Users: {users.filter(u => u.status === 'active').length}</p>
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

### Task Time Tracker Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const [startTime, setStartTime] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval = null;
    if (isTracking) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking]);

  const startTracking = () => {
    setIsTracking(true);
    setStartTime(new Date());
    setSeconds(0);
  };

  const stopTracking = async () => {
    setIsTracking(false);
    const endTime = new Date();
    
    // Save time to backend
    try {
      await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          duration: seconds,
          startTime: startTime.toISOString(),
          endTime: endTime.toISOString()
        })
      });
      alert('Time logged successfully!');
    } catch (error) {
      console.error('Error logging time:', error);
    }
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <h3>Time Tracker</h3>
      <div className="timer-display">{formatTime(seconds)}</div>
      {!isTracking ? (
        <button onClick={startTracking}>Start</button>
      ) : (
        <button onClick={stopTracking}>Stop & Save</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Common Patterns

### Protected Routes with JWT

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }

  // Decode JWT to check role
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    
    if (requiredRole && payload.role !== requiredRole) {
      return <Navigate to="/unauthorized" />;
    }

    return children;
  } catch (error) {
    return <Navigate to="/login" />;
  }
};

// Usage
// <ProtectedRoute requiredRole="admin">
//   <AdminDashboard />
// </ProtectedRoute>
```

### Real-time Notifications

```javascript
const NotificationSystem = () => {
  const [notifications, setNotifications] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    const eventSource = new EventSource(
      `http://localhost:5000/api/notifications/stream?token=${token}`
    );

    eventSource.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      setNotifications(prev => [notification, ...prev]);
    };

    return () => eventSource.close();
  }, [token]);

  return (
    <div className="notifications">
      {notifications.map(notif => (
        <div key={notif.id} className={`notification ${notif.type}`}>
          {notif.message}
        </div>
      ))}
    </div>
  );
};
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your-secret-key-min-32-chars
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### ML Service Configuration

```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
ANOMALY_THRESHOLD=0.85
RISK_THRESHOLD_HIGH=0.7
RISK_THRESHOLD_MEDIUM=0.4
BURNOUT_THRESHOLD=0.75
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check if token is expired
const isTokenExpired = (token) => {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return Date.now() >= payload.exp * 1000;
  } catch {
    return true;
  }
};

// Refresh token if needed
if (isTokenExpired(token)) {
  // Redirect to login or refresh token
  localStorage.removeItem('token');
  window.location.href = '/login';
}
```

### MongoDB Connection Issues

```javascript
// backend/db/connection.js
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
```

### ML Service Not Responding

```python
# Check ML service health
import requests

def check_ml_service():
    try:
        response = requests.get('http://localhost:8000/health')
        if response.status_code == 200:
            print("ML service is healthy")
            return True
    except Exception as e:
        print(f"ML service error: {e}")
        return False
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH']
}));
```

### Performance Optimization

```javascript
// Batch AI predictions for multiple users
const batchPredictRisks = async (userIds) => {
  const response = await fetch('http://localhost:8000/batch-predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ user_ids: userIds })
  });
  return response.json();
};

// Use caching for frequent requests
const cache = new Map();
const getCachedAnalytics = async (userId) => {
  const cacheKey = `analytics-${userId}`;
  if (cache.has(cacheKey)) {
    return cache.get(cacheKey);
  }
  const data = await fetchAnalytics(userId);
  cache.set(cacheKey, data);
  setTimeout(() => cache.delete(cacheKey), 300000); // 5 min cache
  return data;
};
```
