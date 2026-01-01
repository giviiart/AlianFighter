import Phaser from 'phaser';
import axios from 'axios';

class BootScene extends Phaser.Scene {
    preload() {
        // Load assets (replace with your paths)
        this.load.image('player', 'assets/player.png'); // Player sprite
        this.load.image('alien-scout', 'assets/alien-scout.png');
        this.load.image('alien-warrior', 'assets/alien-warrior.png');
        this.load.image('alien-boss', 'assets/alien-boss.png');
        this.load.image('bullet', 'assets/bullet.png');
        this.load.image('powerup', 'assets/powerup.png');
        this.load.image('area51-bg', 'assets/area51-bg.png'); // Desert/base background
        this.load.audio('shoot', 'assets/shoot.wav'); // Sound effect
    }

    create() {
        this.scene.start('GameScene');
    }
}

class GameScene extends Phaser.Scene {
    constructor() {
        super({ key: 'GameScene' });
        this.score = 0;
        this.level = 1;
        this.playerHealth = 100;
        this.enemies = [];
        this.bullets = [];
    }

    create() {
        // Background
        this.add.image(400, 300, 'area51-bg').setScale(2);

        // Player
        this.player = this.physics.add.sprite(100, 300, 'player');
        this.player.setCollideWorldBounds(true);
        this.player.setData('health', this.playerHealth);

        // Controls
        this.cursors = this.input.keyboard.createCursorKeys();
        this.spaceKey = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.SPACE);

        // UI
        this.scoreText = this.add.text(10, 10, 'Score: 0', { fontSize: '20px', fill: '#fff' });
        this.levelText = this.add.text(10, 40, 'Level: 1', { fontSize: '20px', fill: '#fff' });
        this.healthText = this.add.text(10, 70, 'Health: 100', { fontSize: '20px', fill: '#fff' });

        // Generate first level
        this.generateLevel();

        // Shooting
        this.input.on('pointerdown', this.shoot, this);
    }

    update() {
        // Player movement
        if (this.cursors.left.isDown) this.player.setVelocityX(-200);
        else if (this.cursors.right.isDown) this.player.setVelocityX(200);
        else this.player.setVelocityX(0);

        if (this.cursors.up.isDown) this.player.setVelocityY(-200);
        else if (this.cursors.down.isDown) this.player.setVelocityY(200);
        else this.player.setVelocityY(0);

        // Update enemies
        this.enemies.forEach(enemy => this.updateEnemyAI(enemy));

        // Check level completion
        if (this.enemies.length === 0) {
            this.completeLevel();
        }
    }

    async generateLevel() {
        // Simulate Givii AI call (replace with real API)
        try {
            const prompt = `Generate a level for Area 51 Zone ${this.level}: Describe 5-10 alien enemies with types (scout, warrior, boss), positions (x,y from 0-800), and behaviors. Output as JSON.`;
            // Mock response (in production, use axios.post to xAI API)
            const mockResponse = {
                enemies: [
                    { type: 'scout', x: 600, y: 200, behavior: 'patrol' },
                    { type: 'warrior', x: 500, y: 400, behavior: 'chase' },
                    { type: 'boss', x: 700, y: 300, behavior: 'swarm' }
                ]
            };
            // Uncomment for real API: const response = await axios.post('https://api.x.ai/v1/chat/completions', { model: 'grok-1', messages: [{ role: 'user', content: prompt }] });
            // const mockResponse = JSON.parse(response.data.choices[0].message.content);

            mockResponse.enemies.forEach(e => this.spawnAlien(e.type, e.x, e.y, e.behavior));
        } catch (error) {
            console.error('AI generation failed:', error);
            // Fallback: Spawn default enemies
            this.spawnAlien('scout', 600, 200, 'patrol');
        }
    }

    spawnAlien(type, x, y, behavior) {
        const spriteKey = `alien-${type}`;
        const alien = this.physics.add.sprite(x, y, spriteKey);
        alien.setData('type', type);
        alien.setData('behavior', behavior);
        alien.setData('health', type === 'scout' ? 1 : type === 'warrior' ? 3 : 10);
        this.enemies.push(alien);

        // Collision with player
        this.physics.add.collider(alien, this.player, () => {
            this.playerHealth -= 10;
            this.healthText.setText(`Health: ${this.playerHealth}`);
            if (this.playerHealth <= 0) this.scene.restart();
        });
    }

    updateEnemyAI(enemy) {
        const behavior = enemy.getData('behavior');
        if (behavior === 'patrol') {
            // Simple patrol
            this.tweens.add({ targets: enemy, x: enemy.x + 100, duration: 2000, yoyo: true });
        } else if (behavior === 'chase') {
            this.physics.moveToObject(enemy, this.player, 50);
        } else if (behavior === 'swarm') {
            // Swarm: Move in group
            this.physics.moveToObject(enemy, this.player, 30);
        }
    }

    shoot() {
        const bullet = this.physics.add.sprite(this.player.x, this.player.y, 'bullet');
        bullet.setVelocityX(300);
        this.bullets.push(bullet);

        // Sound
        this.sound.play('shoot');

        // Collision with enemies
        this.physics.add.collider(bullet, this.enemies, (bullet, enemy) => {
            enemy.setData('health', enemy.getData('health') - 1);
            bullet.destroy();
            if (enemy.getData('health') <= 0) {
                enemy.destroy();
                this.enemies = this.enemies.filter(e => e !== enemy);
                this.score += 10;
                this.scoreText.setText(`Score: ${this.score}`);
            }
        });
    }

    completeLevel() {
        this.level++;
        this.levelText.setText(`Level: ${this.level}`);
        // Spawn power-up
        const powerup = this.physics.add.sprite(400, 300, 'powerup');
        this.physics.add.overlap(this.player, powerup, () => {
            this.playerHealth += 20;
            this.healthText.setText(`Health: ${this.playerHealth}`);
            powerup.destroy();
            this.generateLevel(); // Next level
        });
    }
}

const config = {
    type: Phaser.AUTO,
    width: 800,
    height: 600,
    parent: 'game-container',
    scene: [BootScene, GameScene],
    physics: { default: 'arcade', arcade: { debug: false } }
};

new Phaser.Game(config);
