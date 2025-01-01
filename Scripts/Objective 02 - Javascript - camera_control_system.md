```
// Camera Control System
const CameraControl = {
    wrapper: null,
    rotationX: -22,  // Camera's default tilt
    rotationY: 0,
    scale: 1,
    STEP: 10,
    SCALE_STEP: 0.15,
    MIN_SCALE: 0.0,
    MAX_SCALE: 3.0,
    
    init() {
        const camera = document.querySelector('.camera');
        if (!camera) return false;
        
        // Create or get wrapper
        if (camera.parentElement.classList.contains('camera-wrapper')) {
            this.wrapper = camera.parentElement;
        } else {
            this.wrapper = document.createElement('div');
            this.wrapper.className = 'camera-wrapper';
            this.wrapper.style.cssText = `
                transform-style: preserve-3d;
                position: absolute;
                width: 100%;
                height: 100%;
                transform-origin: center center;
                transition: transform 0.3s ease;
            `;
            
            // Preserve camera's original position
            const originalTransform = window.getComputedStyle(camera).transform;
            camera.style.transform = originalTransform;
            
            camera.parentNode.insertBefore(this.wrapper, camera);
            this.wrapper.appendChild(camera);
        }
        
        // Add mouse wheel listener
        document.addEventListener('wheel', this.handleMouseWheel.bind(this), { passive: false });
        
        this.updateRotation();
        console.log('Camera control initialized');
        return true;
    },
    
    handleMouseWheel(event) {
        event.preventDefault();
        const zoomDelta = -Math.sign(event.deltaY) * this.SCALE_STEP;
        this.zoom(zoomDelta);
    },
    
    updateRotation() {
        if (!this.wrapper) return;
        
        // Convert angles to radians
        const xRad = this.rotationX * Math.PI / 180;
        const yRad = this.rotationY * Math.PI / 180;
        
        // Pre-calculate trig functions
        const cx = Math.cos(xRad);
        const sx = Math.sin(xRad);
        const cy = Math.cos(yRad);
        const sy = Math.sin(yRad);
        
        // Apply transforms in the correct order:
        // 1. First apply scale
        // 2. Then apply X rotation (tilt)
        // 3. Then rotate around the tilted Y axis
        // 4. Then un-tilt to maintain the same viewing angle
        this.wrapper.style.transform = `
            scale(${this.scale})
            rotateX(${this.rotationX}deg)
            rotateY(${this.rotationY}deg)
            rotateX(${-this.rotationX}deg)
        `;
    },
    
    rotateY(delta) {
        this.rotationY += delta;
        // Normalize angle to -180 to 180 degrees
        if (this.rotationY > 180) this.rotationY -= 360;
        if (this.rotationY < -180) this.rotationY += 360;
        this.updateRotation();
    },
    
    zoom(delta) {
        const newScale = Math.max(this.MIN_SCALE, 
                                Math.min(this.MAX_SCALE, 
                                        this.scale + delta));
        if (newScale !== this.scale) {
            this.scale = newScale;
            this.updateRotation();
        }
    },
    
    reset() {
        this.rotationX = -22;
        this.rotationY = 0;
        this.scale = 1;
        this.updateRotation();
    }
};

// Initialize the control system
CameraControl.init();

// Add keyboard controls with swapped Q and E
document.addEventListener('keydown', (e) => {
    switch(e.key.toLowerCase()) {
        case 'e':  // Swapped from 'q'
            CameraControl.rotateY(-CameraControl.STEP);  // Orbit left
            break;
        case 'q':  // Swapped from 'e'
            CameraControl.rotateY(CameraControl.STEP);   // Orbit right
            break;
        case 'r':
            CameraControl.reset();  // Reset all
            break;
    }
});

console.log('Camera controls ready!');
console.log('--');
console.log('Q: Rotate right');  // Updated
console.log('E: Rotate left');   // Updated
console.log('R: Reset view');
console.log('Mouse Wheel: Zoom in/out');
console.log('--');

```