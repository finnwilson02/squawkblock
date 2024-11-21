# Squawkblock Node Setup Guide

## Raspberry Pi Setup (Trackers)

### 1. Initial Setup
```bash
# Create and activate virtual environment
python3 -m venv ~/drone_venv
source ~/drone_venv/bin/activate

# Add environment activation to .bashrc
echo "source ~/drone_venv/bin/activate" >> ~/.bashrc

# Install required packages
pip install numpy scipy pandas pyzmq msgpack opencv-python scikit-image djitellopy transforms3d bezier shapely filterpy matplotlib flask numba pytest black pylint ipython python-dotenv
```

### 2. Network Permissions
```bash
# Add user to netdev group (requires logout/login to take effect)
sudo usermod -aG netdev $USER
```

### 3. Required Files
Copy from existing Pi:
- `wifi_manager.py`
- `startup.py`

### 4. Command Shortcuts
Add to `~/.bashrc`:
```bash
# Add tracker commands
alias tracker1='cd ~/squawkblock && python3 startup.py tracker1'
alias tracker2='cd ~/squawkblock && python3 startup.py tracker2'
alias tracker3='cd ~/squawkblock && python3 startup.py tracker3'
```

### 5. Apply Changes
```bash
# Reload bashrc
source ~/.bashrc

# Or logout and login for group changes to take effect
```

## Ubuntu Server Setup (Monitor)

### 1. Virtual Environment
```bash
# Create and activate virtual environment
python3 -m venv ~/monitor_venv
source ~/monitor_venv/bin/activate

# Add to .bashrc
echo "source ~/monitor_venv/bin/activate" >> ~/.bashrc
```

### 2. Required Packages
```bash
pip install numpy scipy pandas pyzmq msgpack opencv-python matplotlib
```

### 3. Monitor Script
Save as `monitor.py`:

```python
#!/usr/bin/env python3
import zmq
import json
import cv2
import numpy as np
from threading import Thread
import time
import logging
from pathlib import Path

class DroneMonitor:
    def __init__(self):
        self.context = zmq.Context()
        self.running = False
        self.trackers = {}  # Store latest data from each tracker
        logging.basicConfig(level=logging.INFO)
        self.logger = logging.getLogger("DroneMonitor")
        
        # Set up subscriber for all trackers
        self.subscriber = self.context.socket(zmq.SUB)
        self.subscriber.setsockopt(zmq.SUBSCRIBE, b"video")  # Subscribe to video topics
        
        # Connect to all possible trackers
        for i in range(1, 4):  # trackers 1-3
            tracker = f"tracker{i}.local"
            try:
                self.subscriber.connect(f"tcp://{tracker}:5555")
                self.logger.info(f"Connected to {tracker}")
            except Exception as e:
                self.logger.warning(f"Couldn't connect to {tracker}: {e}")

    def display_loop(self):
        """Display video feeds from all connected trackers."""
        while self.running:
            try:
                # Get message with non-blocking recv
                [topic, telemetry, frame_data] = self.subscriber.recv_multipart()
                telemetry = json.loads(telemetry)
                node = telemetry['node']
                
                # Convert frame data back to image
                frame = cv2.imdecode(
                    np.frombuffer(frame_data, dtype=np.uint8),
                    cv2.IMREAD_COLOR
                )
                
                # Add telemetry overlay
                cv2.putText(frame, f"Battery: {telemetry['battery']}%", 
                           (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
                cv2.putText(frame, f"Height: {telemetry['height']}cm",
                           (10, 70), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
                
                # Store/update frame for this tracker
                self.trackers[node] = frame
                
                # Create combined display of all trackers
                if self.trackers:
                    rows = []
                    max_width = 0
                    row = []
                    
                    for name, frame in sorted(self.trackers.items()):
                        row.append(frame)
                        if len(row) == 2:  # 2 frames per row
                            rows.append(row)
                            max_width = max(max_width, sum(f.shape[1] for f in row))
                            row = []
                    
                    if row:  # Handle any remaining frames
                        rows.append(row)
                        max_width = max(max_width, sum(f.shape[1] for f in row))
                    
                    # Combine frames
                    display = None
                    for row in rows:
                        row_img = np.hstack(row)
                        if display is None:
                            display = row_img
                        else:
                            display = np.vstack([display, row_img])
                    
                    # Show combined display
                    cv2.imshow('Drone Feeds', display)
                    
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
                    
            except Exception as e:
                self.logger.error(f"Display error: {e}")
                time.sleep(0.1)

    def run(self):
        """Main run loop."""
        self.running = True
        self.logger.info("Starting monitor...")
        
        try:
            self.display_loop()
        except KeyboardInterrupt:
            self.logger.info("Shutting down...")
        finally:
            self.cleanup()

    def cleanup(self):
        """Clean up resources."""
        self.running = False
        cv2.destroyAllWindows()
        self.subscriber.close()
        self.context.term()

if __name__ == "__main__":
    monitor = DroneMonitor()
    monitor.run()
```

### 4. Command Shortcut
Add to `~/.bashrc`:
```bash
alias monitor='cd ~/squawkblock && python3 monitor.py'
```

### 5. Apply Changes
```bash
source ~/.bashrc
```

## Usage

### On Tracker Pis
1. Power on Tello drone
2. Wait for drone to create wifi network
3. Run appropriate tracker command:
```bash
tracker1  # On first Pi
tracker2  # On second Pi
tracker3  # On third Pi
```

### On Monitor (Ubuntu Server)
```bash
monitor
```
- Shows all connected drone feeds
- Displays telemetry overlay
- Press 'q' to quit

## Troubleshooting
1. Check Tello wifi is visible:
```bash
nmcli device wifi list | grep TELLO
```

2. Verify ZMQ connections:
```bash
# Test if ports are open
netstat -an | grep 5555
```

3. Test network connectivity:
```bash
# Ping between devices
ping tracker1.local
ping tracker2.local
ping tracker3.local
```

4. Check logs:
```bash
# View recent logs
tail -f ~/tello_logs/tracker1.log
```


Would you like me to add any additional sections to the README or modify any of the scripts?