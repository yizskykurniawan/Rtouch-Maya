# Rtouch 2.0
Object-space translation and rotation tool for Maya with dynamic pivot support.

Preview :

<img width="326" height="593" alt="image" src="https://github.com/user-attachments/assets/64c2b6a6-c17a-4701-a93a-8b5515744c94" />

Script :
```ruby
from maya import cmds
from PySide2 import QtWidgets, QtCore, QtGui

WIN_NAME = "Rtouch_OS_WD"
STEP_DEFAULT = 0.05          # move step
ROT_STEP_DEFAULT = 10.0      # rotation step in degrees
PIVOT_DEFAULT = (0.0786956, 59.107872, 0.809079)  # cm

# Runtime state
_AUTO_PIVOT = True
_CURRENT_PIVOT = None        # latest computed pivot (world space)
_PIVOT_JOB = None           # scriptJob id

def _get_step(field):
    try: return float(field.text())
    except: return STEP_DEFAULT

def _get_rot_step(field):
    try: return float(field.text())
    except: return ROT_STEP_DEFAULT

def _get_pivot(fields):
    def _val(field, fallback):
        try: return float(field.text())
        except: return fallback
    return (_val(fields[0], PIVOT_DEFAULT[0]), _val(fields[1], PIVOT_DEFAULT[1]), _val(fields[2], PIVOT_DEFAULT[2]))

def _format3(v):
    return "{:.6f} {:.6f} {:.6f}".format(v[0], v[1], v[2])

def _compute_dynamic_pivot():
    sel = cmds.ls(sl=True, fl=True) or []
    if not sel: return None
    positions = []
    try:
        verts = cmds.polyListComponentConversion(sel, toVertex=True) or []
        verts = cmds.filterExpand(verts, sm=31) or []
    except: verts = []

    if verts:
        for v in verts:
            try: positions.append(cmds.pointPosition(v, w=True))
            except: pass
    else:
        xforms = [s for s in cmds.ls(sel, type="transform") or []]
        if not xforms:
            parents = list(set(cmds.listRelatives(sel, p=True, type="transform") or []))
            xforms = parents
        for obj in xforms:
            p = None
            try: p = cmds.xform(obj, q=True, ws=True, rp=True)
            except: pass
            if not p:
                try:
                    bb = cmds.exactWorldBoundingBox(obj)
                    p = [(bb[0]+bb[3])/2.0, (bb[1]+bb[4])/2.0, (bb[2]+bb[5])/2.0]
                except: p = None
            if p: positions.append(p)

    if not positions: return None
    count = float(len(positions))
    return (sum(p[0] for p in positions)/count, sum(p[1] for p in positions)/count, sum(p[2] for p in positions)/count)

def nudge_move(dx=0.0, dy=0.0, dz=0.0):
    sel = cmds.ls(sl=True, long=True) or []
    if not sel: cmds.warning("No selection. Select transforms."); return
    for obj in sel:
        try: cmds.move(dx, dy, dz, obj, r=True, os=True, wd=True)
        except Exception as e: cmds.warning(f"Could not move {obj}: {e}")

def nudge_rotate(rx=0.0, ry=0.0, rz=0.0, pivot=None):
    sel = cmds.ls(sl=True, long=True) or []
    if not sel: cmds.warning("No selection. Select transforms/components."); return
    p = _CURRENT_PIVOT if (_AUTO_PIVOT and _CURRENT_PIVOT is not None) else (pivot if pivot is not None else PIVOT_DEFAULT)
    for obj in sel:
        try: cmds.rotate(rx, ry, rz, obj, r=True, os=True, fo=True, p=p)
        except Exception as e: cmds.warning(f"Could not rotate {obj}: {e}")

class RtouchNudgeTool(QtWidgets.QWidget):
    def __init__(self):
        super(RtouchNudgeTool, self).__init__()
        self.setWindowTitle("Rtouch 2.0")
        self.setFixedSize(320, 560)
        self.init_ui()
        self.install_script_job()
        self.update_pivot()

    def init_ui(self):
        layout = QtWidgets.QVBoxLayout(self)
        layout.setSpacing(6)

        # --- Move Step ---
        layout.addWidget(QtWidgets.QLabel("<b>Move Step:</b>"))
        move_step_layout = QtWidgets.QHBoxLayout()
        self.move_step_field = QtWidgets.QLineEdit(str(STEP_DEFAULT))
        btn_reset_move = QtWidgets.QPushButton("Reset")
        btn_reset_move.setFixedWidth(60)
        btn_reset_move.clicked.connect(lambda: self.move_step_field.setText(str(STEP_DEFAULT)))
        move_step_layout.addWidget(self.move_step_field)
        move_step_layout.addWidget(btn_reset_move)
        layout.addLayout(move_step_layout)

        # --- Move Buttons ---
        def make_move_cmd(dx, dy, dz):
            return lambda: nudge_move(dx=self.get_m_step()*dx, dy=self.get_m_step()*dy, dz=self.get_m_step()*dz)

        for axis, n_col, p_col, vals in [
            ("X", "#595973", "#7373bf", (-1, 1)),
            ("Y", "#597359", "#73bf73", (-1, 1)),
            ("Z", "#735959", "#bf7373", (-1, 1))
        ]:
            row = QtWidgets.QHBoxLayout()
            btn_neg = QtWidgets.QPushButton(f"-{axis}")
            btn_neg.setStyleSheet(f"background-color: {n_col}; font-weight: bold; color: white;")
            btn_neg.clicked.connect(make_move_cmd(vals[0] if axis=='X' else 0, vals[0] if axis=='Y' else 0, vals[0] if axis=='Z' else 0))
            
            lbl = QtWidgets.QLabel(axis)
            lbl.setAlignment(QtCore.Qt.AlignCenter)
            lbl.setFixedWidth(40)
            
            btn_pos = QtWidgets.QPushButton(f"+{axis}")
            btn_pos.setStyleSheet(f"background-color: {p_col}; font-weight: bold; color: white;")
            btn_pos.clicked.connect(make_move_cmd(vals[1] if axis=='X' else 0, vals[1] if axis=='Y' else 0, vals[1] if axis=='Z' else 0))
            
            row.addWidget(btn_neg)
            row.addWidget(lbl)
            row.addWidget(btn_pos)
            layout.addLayout(row)

        line = QtWidgets.QFrame(); line.setFrameShape(QtWidgets.QFrame.HLine); layout.addWidget(line)

        # --- Rotate Step ---
        layout.addWidget(QtWidgets.QLabel("<b>Rotate Step:</b>"))
        rot_step_layout = QtWidgets.QHBoxLayout()
        self.rot_step_field = QtWidgets.QLineEdit(str(ROT_STEP_DEFAULT))
        btn_reset_rot = QtWidgets.QPushButton("Reset")
        btn_reset_rot.setFixedWidth(60)
        btn_reset_rot.clicked.connect(lambda: self.rot_step_field.setText(str(ROT_STEP_DEFAULT)))
        rot_step_layout.addWidget(self.rot_step_field)
        rot_step_layout.addWidget(btn_reset_rot)
        layout.addLayout(rot_step_layout)

        # --- Auto Pivot Checkbox ---
        self.auto_cb = QtWidgets.QCheckBox("Auto Pivot from Selection")
        self.auto_cb.setChecked(True)
        self.auto_cb.stateChanged.connect(self.toggle_auto_pivot)
        layout.addWidget(self.auto_cb)

        # --- Manual Pivot Fields ---
        p_lay = QtWidgets.QHBoxLayout()
        self.px = QtWidgets.QLineEdit(str(PIVOT_DEFAULT[0]))
        self.py = QtWidgets.QLineEdit(str(PIVOT_DEFAULT[1]))
        self.pz = QtWidgets.QLineEdit(str(PIVOT_DEFAULT[2]))
        p_lay.addWidget(QtWidgets.QLabel("Px:")); p_lay.addWidget(self.px)
        p_lay.addWidget(QtWidgets.QLabel("Py:")); p_lay.addWidget(self.py)
        p_lay.addWidget(QtWidgets.QLabel("Pz:")); p_lay.addWidget(self.pz)
        layout.addLayout(p_lay)

        # --- Rotate Buttons ---
        def make_rot_cmd(rx, ry, rz):
            return lambda: nudge_rotate(
                rx=self.get_r_step()*rx, ry=self.get_r_step()*ry, rz=self.get_r_step()*rz,
                pivot=(None if _AUTO_PIVOT else _get_pivot([self.px, self.py, self.pz]))
            )

        for axis, n_col, p_col, vals in [
            ("X", "#4c4c80", "#6666cc", (-1, 1)),
            ("Y", "#4c804c", "#66cc66", (-1, 1)),
            ("Z", "#804c4c", "#cc6666", (-1, 1))
        ]:
            row = QtWidgets.QHBoxLayout()
            btn_neg = QtWidgets.QPushButton(f"-{axis}")
            btn_neg.setStyleSheet(f"background-color: {n_col}; font-weight: bold; color: white;")
            btn_neg.clicked.connect(make_rot_cmd(vals[0] if axis=='X' else 0, vals[0] if axis=='Y' else 0, vals[0] if axis=='Z' else 0))
            
            lbl = QtWidgets.QLabel(axis)
            lbl.setAlignment(QtCore.Qt.AlignCenter)
            lbl.setFixedWidth(40)
            
            btn_pos = QtWidgets.QPushButton(f"+{axis}")
            btn_pos.setStyleSheet(f"background-color: {p_col}; font-weight: bold; color: white;")
            btn_pos.clicked.connect(make_rot_cmd(vals[1] if axis=='X' else 0, vals[1] if axis=='Y' else 0, vals[1] if axis=='Z' else 0))
            
            row.addWidget(btn_neg)
            row.addWidget(lbl)
            row.addWidget(btn_pos)
            layout.addLayout(row)

        layout.addStretch()

        # --- Footer ---
        footer = QtWidgets.QLabel("Rtouch Move + Rotate (Object Space)")
        footer.setStyleSheet("font-size: 9px; color: #888;")
        footer.setAlignment(QtCore.Qt.AlignCenter)
        layout.addWidget(footer)

    def get_m_step(self): return _get_step(self.move_step_field)
    def get_r_step(self): return _get_rot_step(self.rot_step_field)

    def toggle_auto_pivot(self, state):
        global _AUTO_PIVOT
        _AUTO_PIVOT = bool(state)
        if _AUTO_PIVOT: self.update_pivot()

    def update_pivot(self):
        global _CURRENT_PIVOT
        if not _AUTO_PIVOT: return
        pv = _compute_dynamic_pivot()
        if pv is None: return
        _CURRENT_PIVOT = pv
        self.px.setText(str(pv[0]))
        self.py.setText(str(pv[1]))
        self.pz.setText(str(pv[2]))
        cmds.inViewMessage(amg=f'Auto Pivot: <hl>{_format3(pv)}</hl>', pos='topLeft', fade=True)

    def install_script_job(self):
        global _PIVOT_JOB
        if _PIVOT_JOB and cmds.scriptJob(exists=_PIVOT_JOB): return
        try:
            _PIVOT_JOB = cmds.scriptJob(event=("SelectionChanged", self.update_pivot), protected=True)
        except: pass

    def closeEvent(self, event):
        global _PIVOT_JOB
        if _PIVOT_JOB and cmds.scriptJob(exists=_PIVOT_JOB):
            cmds.scriptJob(kill=_PIVOT_JOB, force=True)
            _PIVOT_JOB = None
        super(RtouchNudgeTool, self).closeEvent(event)

def show_ui():
    global rtouchNudgeWin
    try:
        rtouchNudgeWin.close()
        rtouchNudgeWin.deleteLater()
    except: pass
    rtouchNudgeWin = RtouchNudgeTool()
    rtouchNudgeWin.show()

show_ui()
```

All code and flow in this page, iterate by Yizsky Kurniawan.

License Creative Commons Attribution. 
(Use for commercial / personal use)
