# RT.BELL Control System - Linux Implementation Guide

## 📋 Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [System Architecture](#system-architecture)
- [Installation Guide](#installation-guide)
- [Configuration](#configuration)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)
- [Security](#security)
- [Backup & Recovery](#backup--recovery)
- [FAQ](#faq)

## Overview

**RT.BELL** is a real-time bell control system designed for Linux environments that automates bell scheduling for schools, offices, and institutions. This implementation uses Linux's native tools and services to provide reliable, automated bell control without requiring additional hardware controllers.

### Key Features
- **Cron-based Scheduling**: Precise bell timing using system cron
- **Multiple Bell Types**: Support for different bell sounds (start/end/emergency)
- **Manual Override**: Physical button support via GPIO
- **Web Dashboard**: React-based management interface
- **Audit Logging**: Complete history of all bell rings
- **Holiday Support**: Automatic scheduling exceptions
- **Offline Operation**: Continues functioning without internet

## Prerequisites

### Hardware Requirements
```bash
# Minimum Requirements
CPU: 1 GHz dual-core
RAM: 2 GB
Storage: 20 GB free space
Network: Ethernet/WiFi connection
Audio: Sound card with speakers
GPIO: For physical button (optional)

# Recommended
CPU: 2 GHz quad-core
RAM: 4 GB
Storage: 40 GB SSD
Network: Static IP configuration
```

### Software Requirements
```bash
# Operating System
- Ubuntu 20.04 LTS or newer
- Debian 11 or newer
- Raspberry Pi OS (for Raspberry Pi)
- CentOS 8/RHEL 8 or newer
- Fedora 34 or newer

# Required Packages
- Node.js 18.x or newer
- npm 9.x or newer
- PostgreSQL 14 or newer
- Redis 6.x or newer
- Nginx (for production)
- Git
```

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│                    LINUX SYSTEM                      │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────┐    ┌─────────────┐                  │
│  │   React     │    │  Node.js    │                  │
│  │  Frontend   │◄──►│   Backend   │                  │
│  │ (Port 3000) │    │ (Port 5000) │                  │
│  └─────────────┘    └──────┬──────┘                  │
│                            │                          │
│  ┌─────────────────────────▼──────────────────────┐  │
│  │              System Services                     │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐         │  │
│  │  │  Cron   │  │ systemd │  │ Socket  │         │  │
│  │  │         │  │         │  │  .io    │         │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘         │  │
│  └───────┼─────────────┼────────────┼───────────────┘  │
│          │             │            │                    │
│  ┌───────▼─────────────▼────────────▼───────────────┐  │
│  │              Hardware Layer                        │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │  │
│  │  │  GPIO   │  │  Audio  │  │ Network │          │  │
│  │  │ Button  │  │  System │  │ Interface│         │  │
│  │  └─────────┘  └────┬────┘  └─────────┘          │  │
│  │                    │                              │  │
│  │              ┌─────▼─────┐                        │  │
│  │              │  Speaker  │                        │  │
│  │              │   (Bell)  │                        │  │
│  │              └───────────┘                        │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Installation Guide

### 1. System Preparation

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y  # Debian/Ubuntu
# OR
sudo dnf update -y  # RHEL/Fedora

# Install essential tools
sudo apt install -y curl wget git build-essential
# OR
sudo dnf install -y curl wget git gcc-c++ make
```

### 2. Install Node.js

```bash
# Using NodeSource repository (Ubuntu/Debian)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version
npm --version
```

### 3. Install PostgreSQL

```bash
# Ubuntu/Debian
sudo apt install -y postgresql postgresql-contrib

# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Secure installation
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'your_secure_password';"
```

### 4. Install Redis

```bash
# Ubuntu/Debian
sudo apt install -y redis-server

# Configure Redis
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Test Redis
redis-cli ping
# Should return: PONG
```

### 5. Clone and Setup Project

```bash
# Create project directory
mkdir -p /opt/rtbell
cd /opt/rtbell

# Clone repository (or create your own)
git clone https://github.com/yourusername/rtbell.git .
# OR create structure manually
mkdir backend frontend scripts logs config sounds
```

### 6. Backend Setup

```bash
cd /opt/rtbell/backend

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express cors dotenv helmet morgan compression \
            @prisma/client socket.io node-cron dayjs \
            bcryptjs jsonwebtoken express-validator \
            nodemailer winston axios

# Install development dependencies
npm install -D typescript @types/node @types/express \
               @types/cors @types/bcryptjs @types/jsonwebtoken \
               prisma ts-node nodemon

# Initialize TypeScript
npx tsc --init

# Initialize Prisma
npx prisma init
```

#### Backend Configuration File (`backend/.env`)
```env
# Server Configuration
PORT=5000
NODE_ENV=production

# Database
DATABASE_URL="postgresql://postgres:your_password@localhost:5432/rtbell"
REDIS_URL="redis://localhost:6379"

# JWT
JWT_SECRET="your_super_secret_jwt_key"
JWT_EXPIRE="7d"

# Email (for notifications)
SMTP_HOST="smtp.gmail.com"
SMTP_PORT=587
SMTP_USER="your_email@gmail.com"
SMTP_PASS="your_app_password"

# Audio Settings
DEFAULT_BELL_SOUND="/opt/rtbell/sounds/bell.mp3"
EMERGENCY_SOUND="/opt/rtbell/sounds/emergency.mp3"
```

#### Database Schema (`backend/prisma/schema.prisma`)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  name      String
  role      Role     @default(ADMIN)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Schedule {
  id          String   @id @default(cuid())
  name        String
  time        String   // Format: "HH:MM"
  days        String[] // ["mon", "tue", "wed", "thu", "fri", "sat", "sun"]
  sound       String   // Path to sound file
  duration    Int      // Duration in seconds
  active      Boolean  @default(true)
  description String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Holiday {
  id          String   @id @default(cuid())
  name        String
  date        DateTime // Specific date
  recurring   Boolean  @default(false) // Annual holiday
  description String?
  createdAt   DateTime @default(now())
}

model BellLog {
  id          String   @id @default(cuid())
  type        LogType  // SCHEDULED, MANUAL, EMERGENCY
  scheduleId  String?
  triggeredAt DateTime @default(now())
  status      Status   // SUCCESS, FAILED
  duration    Int?     // Actual ring duration
  error       String?
}

enum Role {
  ADMIN
  VIEWER
}

enum LogType {
  SCHEDULED
  MANUAL
  EMERGENCY
}

enum Status {
  SUCCESS
  FAILED
}
```

#### Main Backend Server (`backend/server.js`)
```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');
const cron = require('node-cron');
const { createServer } = require('http');
const { Server } = require('socket.io');
const { PrismaClient } = require('@prisma/client');
const redis = require('redis');
const dayjs = require('dayjs');
const isoWeek = require('dayjs/plugin/isoWeek');
const { exec } = require('child_process');
const path = require('path');

dayjs.extend(isoWeek);

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: "http://localhost:3000",
    methods: ["GET", "POST"]
  }
});

const prisma = new PrismaClient();
const redisClient = redis.createClient({ url: process.env.REDIS_URL });

// Middleware
app.use(helmet());
app.use(compression());
app.use(cors());
app.use(express.json());
app.use(morgan('combined'));

// Connect to Redis
redisClient.connect().catch(console.error);

// Bell ringing function
async function ringBell(schedule, type = 'SCHEDULED') {
  try {
    console.log(`🔔 Ringing bell: ${schedule.name} at ${new Date().toISOString()}`);
    
    // Play sound using system audio
    const soundPath = schedule.sound || '/opt/rtbell/sounds/default.mp3';
    exec(`mpg123 "${soundPath}"`, (error, stdout, stderr) => {
      if (error) {
        console.error(`Error playing sound: ${error.message}`);
        return;
      }
    });

    // Log the bell ring
    await prisma.bellLog.create({
      data: {
        type,
        scheduleId: schedule.id,
        status: 'SUCCESS',
        duration: schedule.duration
      }
    });

    // Emit real-time update
    io.emit('bell-rang', {
      schedule: schedule.name,
      time: new Date(),
      type
    });

    return true;
  } catch (error) {
    console.error('Failed to ring bell:', error);
    
    // Log failure
    await prisma.bellLog.create({
      data: {
        type,
        scheduleId: schedule.id,
        status: 'FAILED',
        error: error.message
      }
    });
    
    return false;
  }
}

// Check schedules every minute
cron.schedule('* * * * *', async () => {
  try {
    const now = dayjs();
    const currentTime = now.format('HH:mm');
    const currentDay = now.format('ddd').toLowerCase();
    
    // Check if today is a holiday
    const isHoliday = await prisma.holiday.findFirst({
      where: {
        date: {
          equals: now.startOf('day').toDate()
        }
      }
    });

    if (isHoliday) {
      console.log(`Today is holiday: ${isHoliday.name}`);
      return;
    }

    // Get active schedules for current time and day
    const schedules = await prisma.schedule.findMany({
      where: {
        active: true,
        time: currentTime,
        days: {
          has: currentDay
        }
      }
    });

    // Ring bells for matching schedules
    for (const schedule of schedules) {
      await ringBell(schedule, 'SCHEDULED');
    }
  } catch (error) {
    console.error('Schedule check error:', error);
  }
});

// API Routes
app.get('/api/schedules', async (req, res) => {
  try {
    const schedules = await prisma.schedule.findMany({
      orderBy: { time: 'asc' }
    });
    res.json(schedules);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/schedules', async (req, res) => {
  try {
    const schedule = await prisma.schedule.create({
      data: req.body
    });
    
    io.emit('schedule-updated', schedule);
    res.json(schedule);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.put('/api/schedules/:id', async (req, res) => {
  try {
    const schedule = await prisma.schedule.update({
      where: { id: req.params.id },
      data: req.body
    });
    
    io.emit('schedule-updated', schedule);
    res.json(schedule);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.delete('/api/schedules/:id', async (req, res) => {
  try {
    await prisma.schedule.delete({
      where: { id: req.params.id }
    });
    
    io.emit('schedule-deleted', req.params.id);
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/manual-ring', async (req, res) => {
  try {
    const { scheduleId } = req.body;
    const schedule = await prisma.schedule.findUnique({
      where: { id: scheduleId }
    });
    
    if (!schedule) {
      return res.status(404).json({ error: 'Schedule not found' });
    }
    
    await ringBell(schedule, 'MANUAL');
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/api/logs', async (req, res) => {
  try {
    const logs = await prisma.bellLog.findMany({
      include: { schedule: true },
      orderBy: { triggeredAt: 'desc' },
      take: 100
    });
    res.json(logs);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GPIO Button Handler (for physical button)
const { Gpio } = require('onoff'); // Requires root
if (process.env.ENABLE_GPIO === 'true') {
  const button = new Gpio(17, 'in', 'both');
  
  button.watch(async (err, value) => {
    if (err) {
      console.error('GPIO error:', err);
      return;
    }
    
    if (value === 1) { // Button pressed
      console.log('Manual button pressed');
      
      // Ring emergency bell or configured schedule
      const emergencySchedule = await prisma.schedule.findFirst({
        where: { name: 'EMERGENCY' }
      });
      
      if (emergencySchedule) {
        await ringBell(emergencySchedule, 'EMERGENCY');
      }
      
      io.emit('manual-button-pressed', new Date());
    }
  });
  
  console.log('GPIO button monitoring enabled');
}

// Socket.io connection
io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);
  
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

// Start server
const PORT = process.env.PORT || 5000;
httpServer.listen(PORT, () => {
  console.log(` RT.BELL backend running on port ${PORT}`);
  console.log(` Scheduler started - checking every minute`);
});
```

### 7. Frontend Setup

```bash
cd /opt/rtbell/frontend

# Create Vite + React project
npm create vite@latest . -- --template react

# Install dependencies
npm install axios socket.io-client react-router-dom \
            @reduxjs/toolkit react-redux @mui/material \
            @emotion/react @emotion/styled @mui/icons-material \
            react-hook-form @hookform/resolvers zod \
            dayjs recharts react-toastify

# Install development dependencies
npm install -D tailwindcss postcss autoprefixer \
               @types/react @types/react-dom
```

#### Main Frontend Component (`frontend/src/App.jsx`)
```jsx
import React, { useState, useEffect } from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import io from 'socket.io-client';
import Dashboard from './components/Dashboard';
import ScheduleManager from './components/ScheduleManager';
import BellLogs from './components/BellLogs';
import Settings from './components/Settings';
import Navbar from './components/Navbar';

const socket = io('http://localhost:5000');

const theme = createTheme({
  palette: {
    primary: {
      main: '#1976d2',
    },
    secondary: {
      main: '#dc004e',
    },
  },
});

function App() {
  const [connected, setConnected] = useState(socket.connected);

  useEffect(() => {
    socket.on('connect', () => {
      console.log('Connected to server');
      setConnected(true);
    });

    socket.on('disconnect', () => {
      console.log('Disconnected from server');
      setConnected(false);
    });

    socket.on('bell-rang', (data) => {
      // Show notification
      toast.info(` Bell rang: ${data.schedule}`);
    });

    return () => {
      socket.off('connect');
      socket.off('disconnect');
      socket.off('bell-rang');
    };
  }, []);

  return (
    <ThemeProvider theme={theme}>
      <Router>
        <div className="App">
          <Navbar connected={connected} />
          <Routes>
            <Route path="/" element={<Dashboard socket={socket} />} />
            <Route path="/schedules" element={<ScheduleManager socket={socket} />} />
            <Route path="/logs" element={<BellLogs />} />
            <Route path="/settings" element={<Settings />} />
          </Routes>
          <ToastContainer position="bottom-right" />
        </div>
      </Router>
    </ThemeProvider>
  );
}

export default App;
```

#### Dashboard Component (`frontend/src/components/Dashboard.jsx`)
```jsx
import React, { useState, useEffect } from 'react';
import {
  Grid, Paper, Typography, Box, Card, CardContent,
  Button, LinearProgress
} from '@mui/material';
import {
  AccessTime, NotificationsActive, History,
  PlayArrow, Stop
} from '@mui/icons-material';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, 
         Tooltip, ResponsiveContainer } from 'recharts';
import axios from 'axios';

const Dashboard = ({ socket }) => {
  const [stats, setStats] = useState({
    totalSchedules: 0,
    activeSchedules: 0,
    todayRings: 0,
    systemStatus: 'active'
  });
  const [recentLogs, setRecentLogs] = useState([]);
  const [systemControl, setSystemControl] = useState({
    masterSwitch: true,
    autoMode: true
  });

  useEffect(() => {
    fetchStats();
    fetchRecentLogs();

    socket.on('bell-rang', (data) => {
      // Update stats when bell rings
      fetchStats();
      fetchRecentLogs();
    });

    return () => {
      socket.off('bell-rang');
    };
  }, []);

  const fetchStats = async () => {
    try {
      const response = await axios.get('/api/stats');
      setStats(response.data);
    } catch (error) {
      console.error('Error fetching stats:', error);
    }
  };

  const fetchRecentLogs = async () => {
    try {
      const response = await axios.get('/api/logs?limit=10');
      setRecentLogs(response.data);
    } catch (error) {
      console.error('Error fetching logs:', error);
    }
  };

  const toggleMasterSwitch = async () => {
    try {
      const newState = !systemControl.masterSwitch;
      await axios.post('/api/system/master-switch', { enabled: newState });
      setSystemControl({ ...systemControl, masterSwitch: newState });
    } catch (error) {
      console.error('Error toggling master switch:', error);
    }
  };

  return (
    <Box sx={{ flexGrow: 1, p: 3 }}>
      <Typography variant="h4" gutterBottom>
        Dashboard
      </Typography>

      {/* Status Cards */}
      <Grid container spacing={3} sx={{ mb: 4 }}>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                Total Schedules
              </Typography>
              <Typography variant="h5">
                {stats.totalSchedules}
              </Typography>
              <AccessTime color="primary" />
            </CardContent>
          </Card>
        </Grid>

        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                Active Schedules
              </Typography>
              <Typography variant="h5">
                {stats.activeSchedules}
              </Typography>
              <NotificationsActive color="success" />
            </CardContent>
          </Card>
        </Grid>

        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                Today's Rings
              </Typography>
              <Typography variant="h5">
                {stats.todayRings}
              </Typography>
              <History color="info" />
            </CardContent>
          </Card>
        </Grid>

        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                System Status
              </Typography>
              <Typography variant="h5" sx={{ 
                color: systemControl.masterSwitch ? 'green' : 'red' 
              }}>
                {systemControl.masterSwitch ? 'ACTIVE' : 'PAUSED'}
              </Typography>
              <Box sx={{ mt: 2 }}>
                <Button
                  variant="contained"
                  color={systemControl.masterSwitch ? 'error' : 'success'}
                  startIcon={systemControl.masterSwitch ? <Stop /> : <PlayArrow />}
                  onClick={toggleMasterSwitch}
                  size="small"
                >
                  {systemControl.masterSwitch ? 'Pause All' : 'Resume All'}
                </Button>
              </Box>
            </CardContent>
          </Card>
        </Grid>
      </Grid>

      {/* Schedule Control Panel */}
      <Paper sx={{ p: 3, mb: 4 }}>
        <Typography variant="h6" gutterBottom>
          Schedule Control
        </Typography>
        <Grid container spacing={2}>
          <Grid item xs={12} md={6}>
            <Typography variant="body2" color="textSecondary">
              Master Switch
            </Typography>
            <LinearProgress 
              variant="determinate" 
              value={systemControl.masterSwitch ? 100 : 0}
              color={systemControl.masterSwitch ? 'success' : 'error'}
              sx={{ height: 10, borderRadius: 5, mt: 1 }}
            />
          </Grid>
          <Grid item xs={12} md={6}>
            <Typography variant="body2" color="textSecondary">
              Auto Mode
            </Typography>
            <LinearProgress 
              variant="determinate" 
              value={systemControl.autoMode ? 100 : 0}
              color="primary"
              sx={{ height: 10, borderRadius: 5, mt: 1 }}
            />
          </Grid>
        </Grid>
      </Paper>

      {/* Recent Activity */}
      <Paper sx={{ p: 3 }}>
        <Typography variant="h6" gutterBottom>
          Recent Bell Rings
        </Typography>
        <Box sx={{ height: 300 }}>
          <ResponsiveContainer width="100%" height="100%">
            <LineChart data={recentLogs}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="triggeredAt" />
              <YAxis />
              <Tooltip />
              <Line type="monotone" dataKey="count" stroke="#1976d2" />
            </LineChart>
          </ResponsiveContainer>
        </Box>
      </Paper>
    </Box>
  );
};

export default Dashboard;
```

### 8. Systemd Service Setup

Create systemd service files for automatic startup:

```bash
sudo nano /etc/systemd/system/rtbell-backend.service
```

```ini
[Unit]
Description=RT.BELL Backend Service
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
Type=simple
User=rtbell
Group=rtbell
WorkingDirectory=/opt/rtbell/backend
ExecStart=/usr/bin/node /opt/rtbell/backend/server.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT=5000

# Security
NoNewPrivileges=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectSystem=full
ProtectHome=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo nano /etc/systemd/system/rtbell-frontend.service
```

```ini
[Unit]
Description=RT.BELL Frontend Service
After=network.target

[Service]
Type=simple
User=rtbell
Group=rtbell
WorkingDirectory=/opt/rtbell/frontend
ExecStart=/usr/bin/npm run preview -- --host
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

### 9. Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/rtbell
```

```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Backend API
    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # WebSocket
    location /socket.io/ {
        proxy_pass http://localhost:5000/socket.io/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/rtbell /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 10. Audio Configuration

```bash
# Install audio players
sudo apt install -y mpg123 sox alsa-utils

# Test audio
speaker-test -t sine -f 1000 -l 1

# Create bell sounds directory
mkdir -p /opt/rtbell/sounds

# Download default bell sounds
wget -O /opt/rtbell/sounds/school-bell.mp3 https://example.com/school-bell.mp3
wget -O /opt/rtbell/sounds/emergency.mp3 https://example.com/emergency.mp3
```

### 11. GPIO Button Setup (Optional)

```bash
# Install GPIO library
sudo apt install -y python3-gpiozero python3-rpi.gpio

# For Node.js GPIO support
cd /opt/rtbell/backend
npm install onoff

# Create GPIO monitor script
sudo nano /opt/rtbell/scripts/gpio-monitor.py
```

```python
#!/usr/bin/env python3
import RPi.GPIO as GPIO
import time
import requests
import subprocess

BUTTON_PIN = 17
API_URL = "http://localhost:5000/api/manual-ring"

GPIO.setmode(GPIO.BCM)
GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

def button_callback(channel):
    print("Button pressed!")
    
    # Play immediate feedback sound
    subprocess.run(["mpg123", "/opt/rtbell/sounds/button-press.mp3"])
    
    # Send API request
    try:
        response = requests.post(API_URL, json={"scheduleId": "emergency"})
        if response.status_code == 200:
            print("Bell triggered successfully")
        else:
            print("Failed to trigger bell")
    except Exception as e:
        print(f"Error: {e}")

GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING, 
                     callback=button_callback, bouncetime=300)

print("GPIO monitoring started. Press Ctrl+C to exit.")
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    GPIO.cleanup()
```

```bash
# Create systemd service for GPIO monitor
sudo nano /etc/systemd/system/rtbell-gpio.service
```

```ini
[Unit]
Description=RT.BELL GPIO Button Monitor
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /opt/rtbell/scripts/gpio-monitor.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Configuration

### Environment Variables

Create `/opt/rtbell/backend/.env.production`:
```env
NODE_ENV=production
PORT=5000
DATABASE_URL="postgresql://rtbell:secure_password@localhost:5432/rtbell"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="your_very_secure_jwt_secret_key_here"
JWT_EXPIRE="7d"
ENABLE_GPIO="false"  # Set to "true" if using physical button
```

### Database Initialization

```bash
# Create database
sudo -u postgres createdb rtbell

# Create user
sudo -u postgres createuser rtbell

# Set password
sudo -u postgres psql -c "ALTER USER rtbell WITH PASSWORD 'secure_password';"

# Grant privileges
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE rtbell TO rtbell;"

# Run migrations
cd /opt/rtbell/backend
npx prisma migrate dev --name init
npx prisma generate
```

### Firewall Configuration

```bash
# UFW (Ubuntu/Debian)
sudo ufw allow 22/tcp  # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow 3000/tcp # Frontend dev
sudo ufw allow 5000/tcp # Backend API
sudo ufw enable

# FirewallD (RHEL/Fedora)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=5000/tcp
sudo firewall-cmd --reload
```

## Usage

### Starting the System

```bash
# Start all services
sudo systemctl start rtbell-backend
sudo systemctl start rtbell-frontend
sudo systemctl start rtbell-gpio  # If using button

# Enable auto-start on boot
sudo systemctl enable rtbell-backend
sudo systemctl enable rtbell-frontend
sudo systemctl enable rtbell-gpio

# Check status
sudo systemctl status rtbell-backend
sudo systemctl status rtbell-frontend
```

### Accessing the Dashboard

```
Local Access: http://localhost:3000
Network Access: http://your-server-ip:3000
Production: https://your-domain.com
```

### Managing Schedules

1. Open dashboard in browser
2. Navigate to "Schedules" tab
3. Click "Add Schedule" button
4. Configure:
   - Name (e.g., "Morning Bell")
   - Time (HH:MM format)
   - Days (Monday, Tuesday, etc.)
   - Sound file
   - Duration
5. Click "Save Schedule"

### Manual Control

- **Web Dashboard**: Click "Ring Now" button on any schedule
- **Physical Button**: Press connected GPIO button
- **API**: POST to `/api/manual-ring`

### System Commands

```bash
# View logs
journalctl -u rtbell-backend -f
journalctl -u rtbell-frontend -f

# Test bell manually
curl -X POST http://localhost:5000/api/manual-ring \
  -H "Content-Type: application/json" \
  -d '{"scheduleId":"your-schedule-id"}'

# View database
sudo -u postgres psql -d rtbell -c "SELECT * FROM \"BellLog\" ORDER BY \"triggeredAt\" DESC LIMIT 10;"
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Bell Not Ringing
```bash
# Check audio system
aplay -l
speaker-test -t sine -f 1000 -l 1

# Test sound file directly
mpg123 /opt/rtbell/sounds/bell.mp3

# Check service logs
journalctl -u rtbell-backend -n 50 --no-pager
```

#### 2. Schedules Not Triggering
```bash
# Verify cron is running
sudo systemctl status cron

# Check database has schedules
sudo -u postgres psql -d rtbell -c "SELECT * FROM \"Schedule\" WHERE active=true;"

# Test time matching manually
curl http://localhost:5000/api/test-schedule
```

#### 3. Database Connection Issues
```bash
# Test PostgreSQL connection
sudo -u postgres psql -d rtbell -c "\conninfo"

# Check if PostgreSQL is running
sudo systemctl status postgresql

# Verify credentials in .env
cat /opt/rtbell/backend/.env.production | grep DATABASE
```

#### 4. WebSocket Connection Failed
```bash
# Check if Socket.io is running
curl http://localhost:5000/socket.io/

# Verify firewall rules
sudo ufw status
# OR
sudo firewall-cmd --list-all

# Check Nginx configuration
sudo nginx -t
```

#### 5. Permission Issues
```bash
# Fix ownership
sudo chown -R rtbell:rtbell /opt/rtbell

# Fix sound permissions
sudo usermod -a -G audio rtbell

# Fix GPIO permissions (if using)
sudo usermod -a -G gpio rtbell
```

## Maintenance

### Regular Maintenance Tasks

#### Daily
```bash
# Check system health
./opt/rtbell/scripts/health-check.sh

# View today's logs
curl http://localhost:5000/api/logs?today=true
```

#### Weekly
```bash
# Backup database
pg_dump -U rtbell rtbell > /backup/rtbell-$(date +%Y%m%d).sql

# Rotate logs
sudo logrotate /etc/logrotate.d/rtbell

# Update holiday list
./opt/rtbell/scripts/update-holidays.sh
```

#### Monthly
```bash
# System updates
sudo apt update && sudo apt upgrade -y

# Clean old logs
sudo journalctl --vacuum-time=30d

# Test backup restoration
./opt/rtbell/scripts/test-backup.sh
```

### Monitoring Script

Create `/opt/rtbell/scripts/health-check.sh`:
```bash
#!/bin/bash

echo "=== RT.BELL Health Check ==="
echo "Time: $(date)"
echo ""

# Check backend service
if systemctl is-active --quiet rtbell-backend; then
    echo " Backend: Running"
else
    echo " Backend: STOPPED"
fi

# Check frontend service
if systemctl is-active --quiet rtbell-frontend; then
    echo " Frontend: Running"
else
    echo " Frontend: STOPPED"
fi

# Check PostgreSQL
if systemctl is-active --quiet postgresql; then
    echo " PostgreSQL: Running"
else
    echo " PostgreSQL: STOPPED"
fi

# Check Redis
if systemctl is-active --quiet redis; then
    echo " Redis: Running"
else
    echo " Redis: STOPPED"
fi

# Check disk space
df -h /opt/rtbell | awk 'NR==2 {print "📊 Disk Usage: " $5}'

# Check memory
free -h | awk 'NR==2 {print "📊 Memory: " $3 "/" $2}'

# Check recent errors
echo ""
echo "Recent Errors (last 10 min):"
journalctl -u rtbell-backend --since "10 minutes ago" | grep -i error | tail -5
```

## Security

### SSL Configuration

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Get SSL certificate
sudo certbot --nginx -d your-domain.com

# Auto-renewal
sudo certbot renew --dry-run
```

### Firewall Hardening

```bash
# Allow only necessary ports
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
```

### User Permissions

```bash
# Create dedicated user
sudo useradd -r -s /bin/false -m -d /opt/rtbell rtbell

# Set proper permissions
sudo chown -R rtbell:rtbell /opt/rtbell
sudo chmod 750 /opt/rtbell
sudo chmod 640 /opt/rtbell/backend/.env.production
```

## Backup & Recovery

### Automated Backup Script

Create `/opt/rtbell/scripts/backup.sh`:
```bash
#!/bin/bash

BACKUP_DIR="/backup/rtbell"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
pg_dump -U rtbell rtbell > $BACKUP_DIR/db_$DATE.sql

# Backup configuration
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /opt/rtbell/backend/.env*

# Backup schedules
cp /opt/rtbell/backend/schedules.json $BACKUP_DIR/schedules_$DATE.json

# Backup sounds
tar -czf $BACKUP_DIR/sounds_$DATE.tar.gz /opt/rtbell/sounds

# Keep only last 30 days of backups
find $BACKUP_DIR -type f -mtime +30 -delete

echo "✅ Backup completed: $DATE"
```

### Recovery Procedure

```bash
# Stop services
sudo systemctl stop rtbell-backend

# Restore database
psql -U rtbell rtbell < /backup/rtbell/db_20240101_120000.sql

# Restore config
tar -xzf /backup/rtbell/config_20240101_120000.tar.gz -C /

# Restart services
sudo systemctl start rtbell-backend
```

## FAQ

### General Questions

**Q: Can I use this without internet?**  
A: Yes! The system runs entirely locally. Internet is only needed for remote management.

**Q: What if power fails?**  
A: The system will restart automatically when power returns and continue scheduled bells.

**Q: Can I have multiple bells?**  
A: Yes, you can connect multiple speakers or use different audio outputs.

**Q: How accurate is the timing?**  
A: With NTP sync, accuracy is within milliseconds. Without internet, the system clock maintains time.

**Q: Can I integrate with existing school software?**  
A: Yes, via REST API or webhooks.

### Technical Questions

**Q: What audio formats are supported?**  
A: MP3, WAV, OGG (anything mpg123 or aplay can play).

**Q: How many schedules can I create?**  
A: Unlimited, limited only by system resources.

**Q: Can I set different volumes for different bells?**  
A: Yes, volume can be configured per schedule.

**Q: Does it support holidays?**  
A: Yes, you can mark specific dates or recurring holidays.

**Q: Can I get notifications on missed rings?**  
A: Yes, email/SMS alerts can be configured.

---

## Support

For issues and questions:
- GitHub Issues: https://github.com/yourusername/rtbell/issues
- Email: support@rtbell.com
- Documentation: https://docs.rtbell.com

## License

MIT License - See LICENSE file for details

---

**Built with ❤️ for Linux**
