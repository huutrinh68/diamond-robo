# diamond-robo-gunn
```javascript
public void run() {
    try {
      ErrorLogger.init(this);
      initComponents();
      initColors();

      setAdjustGunForRobotTurn(true);
      setAdjustRadarForGunTurn(true);

      while (true) {
        _gfx.updatePaintProcessing();
        if (!_TC) {
          _move.execute();
        }
        if (!_MC) {
          _gun.execute();
        }
        _radar.execute();
        execute();
      }
    } catch (RuntimeException re) {
      logAndRethrowException(re);
    }
  }
```
