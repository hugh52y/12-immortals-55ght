# 12-immortals-55ght
game thing
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Scrap-Metal Slime Lab</title>
    <script src="https://jsdelivr.net"></script>
    <style>
        body {
            margin: 0;
            background-color: #111;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
        }
        #game-container {
            border: 4px solid #333;
            box-shadow: 0px 10px 30px rgba(0,0,0,0.8);
        }
    </style>
</head>
<body>

    <div id="game-container"></div>

    <script>
        // --- 1. START SCREEN SCENE (Your New Art Asset) ---
        class StartMenu extends Phaser.Scene {
            constructor() {
                super('StartMenu');
            }

            preload() {
                // Loading your 32x32 title screen background
                this.load.image('start_bg', 'start sreen.png');
            }

            create() {
                // Scale your pixel-art backdrop to fill the screen canvas
                let bg = this.add.image(400, 300, 'start_bg');
                bg.setDisplaySize(800, 600);

                // Title Overlay Text
                this.add.text(400, 220, 'SCRAP-METAL SLIME LAB', {
                    fontFamily: 'Courier New',
                    fontSize: '46px',
                    color: '#00ff00',
                    fontStyle: 'bold'
                }).setOrigin(0.5);

                // Prompt to Begin
                let playBtn = this.add.text(400, 420, 'CLICK HERE TO PLAY', {
                    fontFamily: 'Courier New',
                    fontSize: '28px',
                    color: '#ffffff',
                    backgroundColor: '#222222',
                    padding: { x: 20, y: 10 }
                }).setOrigin(0.5).setInteractive();

                // Mouse Hover Effects
                playBtn.on('pointerover', () => playBtn.setStyle({ color: '#00ff00' }));
                playBtn.on('pointerout', () => playBtn.setStyle({ color: '#ffffff' }));

                // Transition to active level scene on click trigger
                playBtn.on('pointerdown', () => {
                    this.scene.start('GamePlay');
                });
            }
        }

        // --- 2. GAMEPLAY SCENE ---
        class GamePlay extends Phaser.Scene {
            constructor() {
                super('GamePlay');
            }

            preload() {
                // Load all your individual Piskel sprite components 
                this.load.spritesheet('boss', 'boss 1 tim tim.png', { frameWidth: 32, frameHeight: 32 });
                this.load.spritesheet('slime_idle', 'slime .png', { frameWidth: 48, frameHeight: 48 });
                this.load.spritesheet('slime_attack', 'slime attack.png', { frameWidth: 32, frameHeight: 32 });
                this.load.spritesheet('rock_death', 'rock death.png', { frameWidth: 32, frameHeight: 32 });
                this.load.image('platform', 'platform.png');
                this.load.image('spike', 'spike.png');
            }

            create() {
                this.cursors = this.input.keyboard.createCursorKeys();
                this.platforms = this.physics.add.staticGroup();
                this.spikes = this.physics.add.staticGroup();
                this.slimes = this.physics.add.group();
                this.bossTime = 0;

                // Build Animations
                this.anims.create({
                    key: 'tim_tim_float',
                    frames: this.anims.generateFrameNumbers('boss', { start: 0, end: 8 }),
                    frameRate: 12,
                    repeat: -1
                });
                this.anims.create({
                    key: 'slime_breathe',
                    frames: this.anims.generateFrameNumbers('slime_idle', { start: 0, end: 8 }),
                    frameRate: 11,
                    repeat: -1
                });
                this.anims.create({
                    key: 'slime_jump',
                    frames: this.anims.generateFrameNumbers('slime_attack', { start: 0, end: 4 }),
                    frameRate: 12,
                    repeat: 0
                });

                // Generate Map Environment 
                for (let x = 16; x < 800; x += 32) {
                    this.platforms.create(x, 584, 'platform');
                }
                this.platforms.create(200, 430, 'platform');
                this.platforms.create(400, 310, 'platform');
                this.platforms.create(600, 430, 'platform');

                this.spikes.create(300, 552, 'spike');
                this.spikes.create(550, 552, 'spike');

                // --- THE PLAYER CHARACTER: BLACK DOT ---
                // Dynamically creating a smooth circular black dot via graphics pipeline
                let dotGraphics = this.make.graphics({ x: 0, y: 0, add: false });
                dotGraphics.fillStyle(0x000000, 1.0);
                dotGraphics.fillCircle(12, 12, 12); // Radius of 12 pixels
                dotGraphics.generateTexture('black_dot', 24, 24);

                this.player = this.physics.add.sprite(100, 450, 'black_dot');
                this.player.setCollideWorldBounds(true);
                this.player.body.setCircle(12); // Circle collision mask

                // Deploy Entities
                let testSlime = this.slimes.create(400, 100, 'slime_idle');
                testSlime.anims.play('slime_breathe');
                testSlime.setCollideWorldBounds(true);
                testSlime.nextAttack = this.time.now + 2000;

                this.boss = this.physics.add.sprite(400, 130, 'boss');
                this.boss.body.allowGravity = false;
                this.boss.anims.play('tim_tim_float');
                this.boss.startY = 130;

                // Physics Overlap System Setup
                this.physics.add.collider(this.player, this.platforms);
                this.physics.add.collider(this.slimes, this.platforms);
                this.physics.add.overlap(this.player, this.spikes, this.hitSpikes, null, this);
                this.physics.add.collider(this.player, this.slimes, this.hitEnemy, null, this);
            }

            update(time, delta) {
                // Controls Matrix
                if (this.cursors.left.isDown) {
                    this.player.setVelocityX(-220);
                } else if (this.cursors.right.isDown) {
                    this.player.setVelocityX(220);
                } else {
                    this.player.setVelocityX(0);
                }

                if (this.cursors.up.isDown && this.player.body.touching.down) {
                    this.player.setVelocityY(-480);
                }

                // Boss Hover Behavior Engine
                this.bossTime += delta * 0.003;
                this.boss.y = this.boss.startY + Math.sin(this.bossTime) * 25;

                // Monster Jump Loop Action Handler
                this.slimes.getChildren().forEach(slime => {
                    if (time > slime.nextAttack && slime.body.touching.down) {
                        slime.anims.play('slime_jump');
                        slime.setVelocityY(-380);
                        let dir = (this.player.x < slime.x) ? -130 : 130;
                        slime.setVelocityX(dir);
                        slime.nextAttack = time + 3000;

                        this.time.delayedCall(1000, () => {
                            if(slime.active) slime.anims.play('slime_breathe');
                        });
                    }
                });
            }

            hitSpikes(player, spike) {
                player.setPosition(100, 450);
                player.setVelocity(0,0);
            }

            hitEnemy(player, enemy) {
                if (player.body.touching.down && enemy.body.touching.up) {
                    enemy.destroy();
                    player.setVelocityY(-250);
                } else {
                    player.setPosition(100, 450);
                }
            }
        }

        // Global Boot Configuration settings
        const config = {
            type: Phaser.AUTO,
            width: 800,
            height: 600,
            parent: 'game-container',
            pixelArt: true,
            physics: {
                default: 'arcade',
                arcade: { gravity: { y: 900 }, debug: false }
            },
            scene: [StartMenu, GamePlay] // StartMenu loads first
        };

        const game = new Phaser.Game(config);
    </script>
</body>
</html>
