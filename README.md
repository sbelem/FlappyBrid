/*
Flappy Bird game under Construction
*/
import java.awt.*;
import java.awt.event.*;
import java.util.ArrayList;
import java.util.Random;
import javax.swing.*;
import java.io.InputStream;
import java.io.BufferedInputStream;
import javax.sound.sampled.AudioInputStream;
import javax.sound.sampled.AudioSystem;
import javax.sound.sampled.Clip;

public class FlappyBird extends JPanel implements ActionListener, KeyListener {

    // Game configuration
    private static final GameConfig CONFIG = new GameConfig();
    private final SoundManager soundManager;

    // Game state
    private Player player;
    private ArrayList<Obstacle> obstacles;
    private Timer gameTimer;
    private Timer obstacleSpawnTimer;
    private boolean isGameActive;
    private double playerScore;
    private final Random randomGenerator;

    // Static configuration class
    private static class GameConfig {

        final int CANVAS_WIDTH = 400;
        final int CANVAS_HEIGHT = 600;
        final int PLAYER_START_X = CANVAS_WIDTH / 6;
        final int PLAYER_START_Y = CANVAS_HEIGHT / 2;
        final int PLAYER_SIZE_WIDTH = 40;
        final int PLAYER_SIZE_HEIGHT = 30;
        final int OBSTACLE_WIDTH = 70;
        final int OBSTACLE_HEIGHT = 480;
        final int OBSTACLE_GAP = CANVAS_HEIGHT / 4;
        final int GAME_SPEED = -5;
        final int JUMP_VELOCITY = -35;
        final double GRAVITY_FORCE = 3;
        final int FRAME_RATE = 60;
        final int OBSTACLE_SPAWN_INTERVAL = 2000;
        final double TERMINAL_VELOCITY = 15.0;

        final Color SCORE_COLOR = new Color(65, 105, 225);
        final Font GAME_FONT = new Font("Helvetica", Font.BOLD, 28);
        final Font MESSAGE_FONT = new Font("Helvetica", Font.PLAIN, 18);
    }

    // Player class
    private class Player {

        int x, y, width, height;
        Image sprite;

        Player(Image sprite) {
            this.sprite = sprite;
            this.width = CONFIG.PLAYER_SIZE_WIDTH;
            this.height = CONFIG.PLAYER_SIZE_HEIGHT;
            reset();
        }

        void reset() {
            x = CONFIG.PLAYER_START_X;
            y = CONFIG.PLAYER_START_Y;
        }
    }

    // Obstacle class
    private class Obstacle {

        int x, y, width, height;
        Image sprite;
        boolean isScored;

        Obstacle(Image sprite, int startX, int startY) {
            this.sprite = sprite;
            this.x = startX;
            this.y = startY;
            this.width = CONFIG.OBSTACLE_WIDTH;
            this.height = CONFIG.OBSTACLE_HEIGHT;
            this.isScored = false;
        }
    }

    // Sound management class
    private class SoundManager {

        private Clip activeClip;

        void playSound(String soundFile) {
            try {
                if (activeClip != null && activeClip.isRunning()) {
                    activeClip.stop();
                }

                InputStream audioSource = getClass().getResourceAsStream(soundFile);
                if (audioSource == null) {
                    System.err.println("Audio file not found: " + soundFile);
                    return;
                }

                BufferedInputStream bufferedAudio = new BufferedInputStream(audioSource);
                AudioInputStream audioStream = AudioSystem.getAudioInputStream(bufferedAudio);
                Clip newClip = AudioSystem.getClip();
                newClip.open(audioStream);
                newClip.start();
                activeClip = newClip;
            } catch (Exception e) {
                System.err.println("Audio playback error: " + e.getMessage());
            }
        }
    }

    private Image backgroundImg;

    public FlappyBird() {
        setPreferredSize(new Dimension(CONFIG.CANVAS_WIDTH, CONFIG.CANVAS_HEIGHT));
        setFocusable(true);
        addKeyListener(this);

        // Initialize game components
        soundManager = new SoundManager();
        randomGenerator = new Random();
        obstacles = new ArrayList<>();

        // Load game assets
        backgroundImg = new ImageIcon(getClass().getResource("./flappybirdbg.png")).getImage();
        Image playerSprite = new ImageIcon(getClass().getResource("./Bunny_b.png")).getImage();
        player = new Player(playerSprite);

        // Initialize timers
        initializeTimers();
    }

    private void initializeTimers() {
        obstacleSpawnTimer = new Timer(CONFIG.OBSTACLE_SPAWN_INTERVAL, e -> spawnObstacle());
        gameTimer = new Timer(1000 / CONFIG.FRAME_RATE, this);
        startGame();
    }

    private void startGame() {
        isGameActive = true;
        playerScore = 0;
        obstacles.clear();
        player.reset();
        obstacleSpawnTimer.start();
        gameTimer.start();
    }

    private void spawnObstacle() {
        if (!isGameActive) {
            return;
        }

        Image topSprite = new ImageIcon(getClass().getResource("./toppipe.png")).getImage();
        Image bottomSprite = new ImageIcon(getClass().getResource("./bottompipe.png")).getImage();

        int randomOffset = randomGenerator.nextInt(CONFIG.CANVAS_HEIGHT / 3);
        int topY = -CONFIG.OBSTACLE_HEIGHT / 4 - randomOffset;

        Obstacle topObstacle = new Obstacle(topSprite, CONFIG.CANVAS_WIDTH, topY);
        Obstacle bottomObstacle = new Obstacle(bottomSprite, CONFIG.CANVAS_WIDTH,
                topY + CONFIG.OBSTACLE_HEIGHT + CONFIG.OBSTACLE_GAP);

        obstacles.add(topObstacle);
        obstacles.add(bottomObstacle);
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        drawGame(g);
    }

    private void drawGame(Graphics g) {
        // Draw background
        g.drawImage(backgroundImg, 0, 0, CONFIG.CANVAS_WIDTH, CONFIG.CANVAS_HEIGHT, null);
        // Draw player
        g.drawImage(player.sprite, player.x, player.y, player.width, player.height, null);

        // Draw obstacles
        for (Obstacle obstacle : obstacles) {
            g.drawImage(obstacle.sprite, obstacle.x, obstacle.y, obstacle.width, obstacle.height, null);
        }

        // Draw score
        g.setColor(CONFIG.SCORE_COLOR);
        g.setFont(CONFIG.GAME_FONT);
        String scoreText = String.format("Score: %d", (int) playerScore);
        g.drawString(scoreText, 10, 35);

        if (!isGameActive) {
            g.setFont(CONFIG.MESSAGE_FONT);
            g.drawString("Press Enter to Play Again", CONFIG.CANVAS_WIDTH / 4, CONFIG.CANVAS_HEIGHT / 3);
        }
    }

    private void updateGameState() {
        if (!isGameActive) {
            return;
        }

        // Update player position
        player.y += CONFIG.GRAVITY_FORCE;
        player.y = Math.max(0, player.y);

        // Update obstacles
        for (int i = obstacles.size() - 1; i >= 0; i--) {
            Obstacle obstacle = obstacles.get(i);
            obstacle.x += CONFIG.GAME_SPEED;

            // Score counting
            if (!obstacle.isScored && obstacle.x + obstacle.width < player.x) {
                obstacle.isScored = true;
                playerScore += 0.5;
            }

            // Collision detection
            if (checkCollision(player, obstacle)) {
                endGame();
                return;
            }

            // Remove off-screen obstacles
            if (obstacle.x + obstacle.width < 0) {
                obstacles.remove(i);
            }
        }

        // Check if player has fallen
        if (player.y > CONFIG.CANVAS_HEIGHT) {
            endGame();
        }
    }

    private boolean checkCollision(Player player, Obstacle obstacle) {
        return player.x < obstacle.x + obstacle.width
                && player.x + player.width > obstacle.x
                && player.y < obstacle.y + obstacle.height
                && player.y + player.height > obstacle.y;
    }

    private void endGame() {
        isGameActive = false;
        soundManager.playSound("./colission.wav");
        obstacleSpawnTimer.stop();
        gameTimer.stop();
    }

    @Override
    public void actionPerformed(ActionEvent e) {
        updateGameState();
        repaint();
    }

    @Override
    public void keyPressed(KeyEvent e) {
        if (e.getKeyCode() == KeyEvent.VK_ENTER) {
            if (!isGameActive) {
                startGame();
            }
        } else if (e.getKeyCode() == KeyEvent.VK_UP || e.getKeyCode() == KeyEvent.VK_SPACE) {
            if (isGameActive) {
                player.y += CONFIG.JUMP_VELOCITY;
                soundManager.playSound("./flappingsound.wav");
            }
        }
    }

    @Override
    public void keyTyped(KeyEvent e) {
    }

    @Override
    public void keyReleased(KeyEvent e) {
    }
}
