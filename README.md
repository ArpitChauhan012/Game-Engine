// GameEngine.java
// Author: Arpit Chauhan
// Roll No: 2401330130062

import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.ArrayList;
import java.util.Random;

public class GameEngine {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(GameEngine::createAndShowGUI);
    }

    private static void createAndShowGUI() {
        JFrame frame = new JFrame("Simple Game Engine Demo");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(900, 600);
        frame.setLayout(new BorderLayout(8, 8));

        GamePanel gamePanel = new GamePanel();
        frame.add(gamePanel, BorderLayout.CENTER);

        // Control panel (left)
        JPanel control = new JPanel();
        control.setLayout(new GridLayout(0, 1, 8, 8));
        control.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

        JButton startBtn = new JButton("Start");
        JButton pauseBtn = new JButton("Pause");
        JButton spawnBtn = new JButton("Spawn Random");
        JButton clearBtn = new JButton("Clear");
        JButton gravityToggle = new JButton("Toggle Gravity");
        control.add(startBtn);
        control.add(pauseBtn);
        control.add(spawnBtn);
        control.add(clearBtn);
        control.add(gravityToggle);

        startBtn.addActionListener(e -> gamePanel.start());
        pauseBtn.addActionListener(e -> gamePanel.pause());
        spawnBtn.addActionListener(e -> gamePanel.spawnRandom());
        clearBtn.addActionListener(e -> gamePanel.clearObjects());
        gravityToggle.addActionListener(e -> gamePanel.toggleGravity());

        // Debug / Info Panel (right)
        JTextArea debug = new JTextArea();
        debug.setEditable(false);
        debug.setFont(new Font("Monospaced", Font.PLAIN, 12));
        JScrollPane debugScroll = new JScrollPane(debug);
        debugScroll.setPreferredSize(new Dimension(260, 0));

        // Footer with author info
        JLabel footer = new JLabel("Created by Arpit Chauhan | Roll No: 2401330130062", JLabel.CENTER);

        frame.add(control, BorderLayout.WEST);
        frame.add(debugScroll, BorderLayout.EAST);
        frame.add(footer, BorderLayout.SOUTH);

        // Hook the debug updater
        gamePanel.setDebugArea(debug);

        frame.setLocationRelativeTo(null);
        frame.setVisible(true);
    }
}

// =================== GamePanel: rendering + physics + input ===================
class GamePanel extends JPanel implements ActionListener, KeyListener, MouseListener {
    private final java.util.List<GameObject> objects = new ArrayList<>();
    private final Timer timer;
    private boolean running = false;
    private double gravity = 800.0; // px/s^2
    private final double timeStep = 0.016; // seconds per tick (~60 FPS)
    private final Random rand = new Random();
    private JTextArea debugArea;
    private int selectedIndex = -1;

    // player-like controllable object index (optional)
    private int playerIndex = -1;

    GamePanel() {
        setBackground(Color.white);
        setFocusable(true);
        addKeyListener(this);
        addMouseListener(this);

        // pre-spawn a player object
        spawnPlayer();

        timer = new Timer((int)(timeStep * 1000), this);
    }

    void setDebugArea(JTextArea area) {
        this.debugArea = area;
    }

    void start() {
        if (!running) {
            timer.start();
            running = true;
            requestFocusInWindow();
        }
    }

    void pause() {
        if (running) {
            timer.stop();
            running = false;
        }
    }

    void toggleGravity() {
        gravity = (gravity == 0.0) ? 800.0 : 0.0;
    }

    void spawnPlayer() {
        // spawn a controllable "player" near top-left
        GameObject p = new GameObject(80, 80, 20, true);
        p.color = new Color(30, 144, 255);
        objects.add(p);
        playerIndex = objects.size() - 1;
        selectedIndex = playerIndex;
    }

    void spawnRandom() {
        int x = 100 + rand.nextInt(600);
        int y = 30 + rand.nextInt(80);
        int r = 10 + rand.nextInt(28);
        GameObject o = new GameObject(x, y, r, false);
        o.vx = rand.nextDouble() * 200 - 100;
        o.vy = rand.nextDouble() * 50;
        o.color = new Color(rand.nextInt(200), rand.nextInt(200), rand.nextInt(200));
        objects.add(o);
    }

    void clearObjects() {
        objects.clear();
        spawnPlayer();
        playerIndex = 0;
        selectedIndex = 0;
        repaint();
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        // draw ground line
        int groundY = getHeight() - 40;
        g.setColor(new Color(60, 60, 60));
        g.fillRect(0, groundY, getWidth(), 40);

        // draw objects
        for (int i = 0; i < objects.size(); i++) {
            GameObject o = objects.get(i);
            o.draw((Graphics2D) g);
            if (i == selectedIndex) {
                // highlight selected object
                g.setColor(Color.red);
                g.drawRect((int)(o.x - o.r) - 2, (int)(o.y - o.r) - 2, (int)(o.r * 2) + 4, (int)(o.r * 2) + 4);
            }
        }

        // small instructions overlay
        g.setColor(Color.black);
        g.setFont(new Font("SansSerif", Font.PLAIN, 12));
        g.drawString("Keys: ← → to move player, SPACE to jump | Click to select/spawn", 10, 18);
        g.drawString("Objects: " + objects.size(), 10, 36);
    }

    // game loop tick
    @Override
    public void actionPerformed(ActionEvent e) {
        updatePhysics();
        repaint();
        updateDebug();
    }

    private void updatePhysics() {
        int groundY = getHeight() - 40;
        for (int i = 0; i < objects.size(); i++) {
            GameObject o = objects.get(i);

            // integrate velocities (simple Euler)
            if (!o.fixed) {
                o.vy += gravity * timeStep;
            }

            o.x += o.vx * timeStep;
            o.y += o.vy * timeStep;

            // boundary checks: left/right walls
            if (o.x - o.r < 0) {
                o.x = o.r;
                o.vx = -o.vx * o.restitution;
            } else if (o.x + o.r > getWidth()) {
                o.x = getWidth() - o.r;
                o.vx = -o.vx * o.restitution;
            }

            // floor collision
            if (o.y + o.r > groundY) {
                o.y = groundY - o.r;
                if (!o.fixed) {
                    o.vy = -o.vy * o.restitution;
                    // small damping to settle quicker
                    o.vx *= 0.98;
                    // tiny threshold to stop bouncing forever
                    if (Math.abs(o.vy) < 40) o.vy = 0;
                } else {
                    o.vy = 0;
                }
            }
        }

        // simple pairwise collision resolution (circle vs circle)
        for (int i = 0; i < objects.size(); i++) {
            for (int j = i + 1; j < objects.size(); j++) {
                GameObject a = objects.get(i);
                GameObject b = objects.get(j);
                resolveCircleCollision(a, b);
            }
        }
    }

    private void resolveCircleCollision(GameObject a, GameObject b) {
        double dx = b.x - a.x;
        double dy = b.y - a.y;
        double dist = Math.hypot(dx, dy);
        double minDist = a.r + b.r;
        if (dist > 0 && dist < minDist) {
            // push them apart
            double overlap = minDist - dist + 0.01;
            double nx = dx / dist;
            double ny = dy / dist;
            double totalMass = a.mass + b.mass;
            double ratioA = b.mass / totalMass;
            double ratioB = a.mass / totalMass;

            if (!a.fixed) {
                a.x -= nx * overlap * ratioA;
                a.y -= ny * overlap * ratioA;
            }
            if (!b.fixed) {
                b.x += nx * overlap * ratioB;
                b.y += ny * overlap * ratioB;
            }

            // approximate elastic collision response on velocity along normal
            double relVel = (b.vx - a.vx) * nx + (b.vy - a.vy) * ny;
            double e = Math.min(a.restitution, b.restitution);
            double j = -(1 + e) * relVel;
            j /= (1.0 / a.mass) + (1.0 / b.mass);

            if (!a.fixed) {
                a.vx -= (j / a.mass) * nx;
                a.vy -= (j / a.mass) * ny;
            }
            if (!b.fixed) {
                b.vx += (j / b.mass) * nx;
                b.vy += (j / b.mass) * ny;
            }
        }
    }

    private void updateDebug() {
        if (debugArea == null) return;
        StringBuilder sb = new StringBuilder();
        sb.append("Objects: ").append(objects.size()).append("\n");
        sb.append(String.format("Gravity: %.1f px/s^2\n", gravity));
        sb.append("Time step: ").append(timeStep).append("s\n\n");

        if (selectedIndex >= 0 && selectedIndex < objects.size()) {
            GameObject s = objects.get(selectedIndex);
            sb.append("Selected Object:\n");
            sb.append(String.format("  Pos: (%.1f, %.1f)\n", s.x, s.y));
            sb.append(String.format("  Vel: (%.1f, %.1f)\n", s.vx, s.vy));
            sb.append(String.format("  Radius: %.1f\n", s.r));
            sb.append("  Fixed: ").append(s.fixed).append("\n");
            sb.append("  Restitution: ").append(s.restitution).append("\n");
        } else {
            sb.append("No selection\n");
        }

        debugArea.setText(sb.toString());
    }

    // =================== Input Handling ===================
    @Override
    public void keyTyped(KeyEvent e) {}

    @Override
    public void keyPressed(KeyEvent e) {
        if (playerIndex >= 0 && playerIndex < objects.size()) {
            GameObject p = objects.get(playerIndex);
            if (e.getKeyCode() == KeyEvent.VK_LEFT) {
                p.vx = -220;
            } else if (e.getKeyCode() == KeyEvent.VK_RIGHT) {
                p.vx = 220;
            } else if (e.getKeyCode() == KeyEvent.VK_SPACE) {
                // allow jump only if near ground
                int groundY = getHeight() - 40;
                if (Math.abs((p.y + p.r) - groundY) < 6) {
                    p.vy = -420;
                }
            } else if (e.getKeyCode() == KeyEvent.VK_C) {
                // toggle selection forward
                selectedIndex++;
                if (selectedIndex >= objects.size()) selectedIndex = 0;
            }
        }
    }

    @Override
    public void keyReleased(KeyEvent e) {
        if (playerIndex >= 0 && playerIndex < objects.size()) {
            GameObject p = objects.get(playerIndex);
            if (e.getKeyCode() == KeyEvent.VK_LEFT || e.getKeyCode() == KeyEvent.VK_RIGHT) {
                // stop horizontal player movement when key released
                p.vx = 0;
            }
        }
    }

    @Override
    public void mouseClicked(MouseEvent e) {
        // left click selects an object if clicked near; right click spawns a new object
        if (SwingUtilities.isLeftMouseButton(e)) {
            int idx = findObjectAt(e.getX(), e.getY());
            if (idx >= 0) {
                selectedIndex = idx;
            } else {
                // spawn a small object at click
                GameObject o = new GameObject(e.getX(), e.getY(), 12 + rand.nextInt(18), false);
                o.vx = rand.nextDouble() * 120 - 60;
                o.vy = -40;
                objects.add(o);
                selectedIndex = objects.size() - 1;
            }
        } else if (SwingUtilities.isRightMouseButton(e)) {
            // spawn random big object
            GameObject o = new GameObject(e.getX(), e.getY(), 20 + rand.nextInt(36), false);
            o.vx = rand.nextDouble() * 200 - 100;
            objects.add(o);
            selectedIndex = objects.size() - 1;
        }
    }

    private int findObjectAt(int x, int y) {
        for (int i = objects.size() - 1; i >= 0; i--) {
            GameObject o = objects.get(i);
            double dx = x - o.x;
            double dy = y - o.y;
            if (dx * dx + dy * dy <= o.r * o.r) return i;
        }
        return -1;
    }

    @Override public void mousePressed(MouseEvent e) {}
    @Override public void mouseReleased(MouseEvent e) {}
    @Override public void mouseEntered(MouseEvent e) {}
    @Override public void mouseExited(MouseEvent e) {}
}

// =================== Simple GameObject ===================
class GameObject {
    double x, y;      // position (px)
    double vx = 0, vy = 0; // velocity (px/s)
    double r;         // radius (px)
    double mass;
    double restitution = 0.6; // bounce factor
    boolean fixed = false; // if true: not affected by gravity
    Color color = Color.gray;

    GameObject(double x, double y, double r, boolean fixed) {
        this.x = x;
        this.y = y;
        this.r = r;
        this.fixed = fixed;
        this.mass = Math.max(1.0, r * 0.2);
    }

    void draw(Graphics2D g2) {
        g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        g2.setColor(color);
        g2.fillOval((int)(x - r), (int)(y - r), (int)(r * 2), (int)(r * 2));
        g2.setColor(Color.black);
        g2.drawOval((int)(x - r), (int)(y - r), (int)(r * 2), (int)(r * 2));
    }
}
