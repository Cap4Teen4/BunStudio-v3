<!DOCTYPE html>



<style>
    @keyframes ripple {
        to {
            transform: scale(4);
            opacity: 0;
        }
    }

</style></head>



    <div style="
    background:#11111100;
    padding:20px;
    border-radius:12px;
    ">
    <div class="section-title flip-in visible">
        <h2><span style="font-size:30px;color:#ffffff">3D Printer Cam:  </span></h2> <span style="font-size:30px;color:#7f16f7"> Creality K2 Plus</span><br>
        <h2><span style="font-size:30px;color:#ffffff">Status: </span></h2><span style="font-size:30px;color:#fc2c2c"> Live </span><br>
    </div>

    <iframe 
        src="http://10.0.0.73:4408/camera.html"
        style="
        width:100%;
        aspect-ratio:16/9;
        border:none;
        border-radius:10px;
        box-shadow:0 0 20px #00ffff;
        ">
    </iframe>
    </div>
    <section class="gcode-preview gcode-preview--wide" id="gcode-preview"></section>

<div class="mainsail-card">
    <div class="card-header">
        <span>Gcode Vector Preview (SVG Method)</span>
    </div>
    <div class="card-body">
        <div class="controls">
            <label>Target Preview Layer:</label>
            <input type="number" id="layer-num" value="12" min="1" style="width: 50px; background: #272727; color: #fff; border: 1px solid #444; padding: 4px; border-radius: 4px;">
            
            <!-- Added a fallback file input right inside your layout -->
            <input type="file" id="gcode-file-picker" accept=".gcode,.txt" style="font-size: 12px; max-width: 180px; color: #aaa;">
            
            <button id="render-btn" class="mainsail-btn">Generate SVG View</button>
        </div>
        
        <!-- Live Scalable Container -->
        <div id="svg-render-container" class="visualizer-container">
            <span style="color: #666; font-size: 14px;">Select a local file or place a G-code in your server assets folder to begin.</span>
        </div>
    </div>
</div>

<style>
    .mainsail-card { background: #1e1e1e; color: #fff; font-family: sans-serif; border-radius: 8px; max-width: 650px; margin: 20px auto; border: 1px solid #333; }
    .card-header { padding: 12px; background: rgba(255,255,255,0.05); font-weight: 500; border-bottom: 1px solid #2d2d2d; }
    .card-body { padding: 16px; }
    .controls { display: flex; gap: 10px; align-items: center; margin-bottom: 15px; flex-wrap: wrap; }
    .visualizer-container { background: #111; border: 1px solid #222; border-radius: 6px; height: 450px; padding: 10px; display: flex; align-items: center; justify-content: center; overflow: hidden; }
    .visualizer-container svg { width: 100%; height: 100%; }
    .mainsail-btn { background: #1976d2; color: #fff; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: 500; }
    .mainsail-btn:hover { background: #1565c0; }
</style>

<script>
    // Config: Point this exactly to your local K2 Plus Moonraker backend environment
    const PRINTER_IP = "10.0.0.73"; 
    const FLUIDD_PORT = "4408"; 

    // Raw text parser function
    function convertGCodeToSvg(gcodeText, targetLayer = 1) {
        const lines = gcodeText.split('\n');
        let svgPaths = "";
        let currentX = 0, currentY = 0, currentE = 0;
        let currentLayer = 0;
        let isExtruding = false;
        
        let minX = 999, minY = 999, maxX = -999, maxY = -999;

        for (let line of lines) {
            line = line.trim();
            
            if (line.startsWith(';LAYER:')) {
                currentLayer = parseInt(line.split(':')[1]) || currentLayer;
            } else if (line.startsWith('; BEFORE_LAYER_CHANGE')) {
                currentLayer++;
            }

            if (currentLayer !== targetLayer) continue;

            if (line.startsWith('G1') || line.startsWith('G0')) {
                const xMatch = line.match(/X([\d.]+)/);
                const yMatch = line.match(/Y([\d.]+)/);
                const eMatch = line.match(/E([\d.-]+)/);

                let newX = xMatch ? parseFloat(xMatch[1]) : currentX;
                let newY = yMatch ? parseFloat(yMatch[1]) : currentY;
                let newE = eMatch ? parseFloat(eMatch[1]) : currentE;
                
                isExtruding = eMatch && (newE > currentE);

                if (xMatch || yMatch) {
                    if (newX < minX) minX = newX; if (newX > maxX) maxX = newX;
                    if (newY < minY) minY = newY; if (newY > maxY) maxY = newY;

                    if (isExtruding) {
                        svgPaths += `<path d="M ${currentX} ${currentY} L ${newX} ${newY}" stroke="#1976d2" stroke-width="0.4" fill="none" />\n`;
                    } else {
                        svgPaths += `<path d="M ${currentX} ${currentY} L ${newX} ${newY}" stroke="#555555" stroke-width="0.1" stroke-dasharray="1,1" fill="none" />\n`;
                    }

                    currentX = newX;
                    currentY = newY;
                }
                currentE = newE; 
            }
        }

        const width = (maxX - minX) || 220;
        const height = (maxY - minY) || 220;
        
        return `
            <svg viewBox="${minX - 5} ${minY - 5} ${width + 10} ${height + 10}" xmlns="http://w3.org">
                <g transform="scale(1, -1) translate(0, -${minY + maxY})"> 
                    ${svgPaths}
                </g>
            </svg>
        `;
    }

    // Main API Execution Hook
    document.getElementById('render-btn').addEventListener('click', async () => {
        const targetLayer = parseInt(document.getElementById('layer-num').value);
        const container = document.getElementById('svg-render-container');
        const fileInput = document.getElementById('gcode-file-picker');
        
        container.innerHTML = "<span style='color:#aaa;'>Polling printer API endpoints...</span>";

        // Fallback: If user uploads a file manually, prioritize it
        if (fileInput && fileInput.files.length > 0) {
            const localFile = fileInput.files[0];
            const fileReader = new FileReader();
            fileReader.onload = function(e) {
                container.innerHTML = convertGCodeToSvg(e.target.result, targetLayer);
            };
            fileReader.readAsText(localFile);
            return;
        }

        // Live Automated Path: Requesting metadata stream from K2 Plus
        try {
            // 1. Query the print_stats object to get the active filename
            const statusResponse = await fetch(`http://${PRINTER_IP}:${FLUIDD_PORT}/printer/objects/query?print_stats`);
            if (!statusResponse.ok) throw new Error("Could not talk to K2 Plus server.");
            
            const statusData = await statusResponse.json();
            const filename = statusData.result.status.print_stats.filename;

            if (!filename) {
                container.innerHTML = "<span style='color:#orange;'>Printer is currently idle (No active print file found).</span>";
                return;
            }

            container.innerHTML = `<span style='color:#aaa;'>Downloading active path file: ${filename}...</span>`;

            // 2. Stream the raw text from the internal virtual SD Card 
            const fileResponse = await fetch(`http://${PRINTER_IP}:${FLUIDD_PORT}/server/files/gcodes/${encodeURIComponent(filename)}`);
            if (!fileResponse.ok) throw new Error("Could not download G-code from printer virtual storage.");
            
            const gcodeRawText = await fileResponse.text();

            // 3. Render vectors
            const computedSvgCode = convertGCodeToSvg(gcodeRawText, targetLayer);
            container.innerHTML = computedSvgCode;

        } catch (err) {
            console.error(err);
            container.innerHTML = `
                <span style='color:#ef5350; text-align:center; font-size:13px;'>
                    Error connecting directly to K2 Plus at ${PRINTER_IP}.<br>
                    <span style="color:#888; font-size:11px; display:block; margin-top:4px;">
                        Ensure the printer is turned on and that your website has permission to view it inside moonraker.conf.
                    </span>
                </span>`;
        }
    });
</script>





    
    <!-- Hero Section 
    <section class="hero" id="home" role="banner">
        <div class="hero-slider">
            <div class="hero-slide active"></div>
            <div class="hero-slide"></div>
            <div class="hero-slide"></div>
            <div class="hero-slide"></div>
        </div>
        <div class="floating-elements">
            <div class="floating-element"></div>
            <div class="floating-element"></div>
            <div class="floating-element"></div>
        </div>
        <div class="hero-content">

            <h1 class="hero-title">
                <span class="typing-text" data-text="Midnight Bun Studios"> </span>
            </h1>
            <div class="hero-subtitles">🧑‍💻 | Fan Made Studio Projects</div>
            <div class="hero-subtitles">🍪 | STUDIO STATUS ✅ - BETA TESTING </div>
            <h2 class="hero-subtitles">Current Focus | Building the Team</h2>
            <h2 class="hero-subtitles">Current Projects | The Isle OCR Tracker</h2>
            <p class="hero-description">
            FIND YOUR FRIENDS IN-GAME<br />
            HOW IT WORKS:<br />
            Step 1. <br />
            Enter your in-game name and click Create.<br />
            Step 2. <br />
            Share the 6-letter room code with friends.<br />
            Step 3. <br />
            Click Share Screen and select your game window.<br />
            Step 4. <br />
            Press Tab — waypoints automatically appear.<br />
            <br />
            <big>If you Need assistance Ask Staff Members</big>
            
            </p>
            <div class="hero-buttons">
                <a href="#how-it-works" class="btn btn-dark cta-btn-primary" target="_blank" rel="noopener noreferrer" aria-label="Join our Discord server to get started">
                    <i class="fab fa-discord" aria-hidden="true"></i>
                        <span>Get Started</span>
                    <div class="btn-shine" aria-hidden="true"></div>
                </a>
                    <a href="#team" class="btn btn-secondary btn-secondary" aria-label="Learn more about how to get started">
                        <i class="fas fa-info-circle" aria-hidden="true"></i>
                        <span>Meet The Team</span>
                        <div class="btn-shine" aria-hidden="true"></div>
                    </a>
            </div>
        </div>
    </section>


    


    <div class="feature-card feature-card--required zoom-in visible">
        <div class="feature-card__content">

            <div class="feature-card__left">
                <div class="feature-icon">
                    <img src="assets/img/blue-gate-confiscation-room-key.png" alt="Contributors">
                </div>
            </div>

            <div class="feature-card__right">
                <h3 class="feature-title"><span style="font-size:38px;color:#95efff">Looking For Contributor Roles:</span></h3>

                <p class="feature-description">
                    <span style="font-size:18px;color:#ffffff">
                    Join our Discord to follow updates!<br> 
                    Feel free to ask questions, and collaborate with the team.</span>
                </p>

                <p class="feature-description">
                    <span style="font-size:18px;color:#ffffff">We are currently looking for:<br></span>
                    <strong><span style="font-size:18px;color:#ff3636"> Staff Members | Model Creators | Developers | Designers | Artists</span></strong>
                </p>

                <a href="#how-it-works"
                   class="btn btn-dark cta-btn-primary"
                   target="_blank"
                   rel="noopener noreferrer">

                    <i class="fab fa-discord"></i>
                    <span>Join Discord</span>
                    <div class="btn-shine"></div>
                </a>
            </div>

        </div>
    </div>
</section>
    
</div>

    <section class="team" id="team">
        <div class="team-container">
            <div class="section-title flip-in visible">
                <h2><span style="font-size:30px;color:#ffffff">Meet Current The Team:</span></h2>
                <p><span style="font-size:18px;color:#ffffff">The dedicated <br>Developers / Admins / Staff Members / Model Creator / Project Lead<br> involved in Starting / Activly Developing Midnight Bun Studios<br><br> Currently we are a passion Project team, Trying to make a games more User-Friendly!</span></p>
            </div>
            <div class="team-grid">
                <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/42Oreobun.png" alt="42Oreobun">
                    </div>
                    <h3 class="team-name">42Oreobun</h3>
                    <p class="team-role">Owner / Developer / Model Creator / Project Lead</p>
                </div>
                <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/BunnyOreo.png" alt="BunnyOreo">
                    </div>
                    <h3 class="team-name">BunnyOreo</h3>
                    <p class="team-role">Co-Owner / Server Admin</p>
                </div>   
                <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/yukon.webp" alt="Chemical OverDose">
                    </div>
                    <h3 class="team-name">Chemical OverDose</h3>
                    <p class="team-role">Server Admin</p>
                </div>
                <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/kett.webp" alt="Butternuts">
                    </div>
                    <h3 class="team-name">Butternuts</h3>
                    <p class="team-role">Server Moderator</p>
                </div>
                 <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/trolley.webp" alt="TrolleyRolley">
                    </div>
                    <h3 class="team-name">TrolleyRolley</h3>
                    <p class="team-role">Server Admin</p>
                </div>
                <div class="team-card scale-up visible">
                    <div class="team-image">
                        <img src="assets/img/ugly.webp" alt="UglyFiveTwo">
                    </div>
                    <h3 class="team-name">UglyFiveTwo</h3>
                    <p class="team-role">Server Admin</p>
                </div>
           </div>
        </div>
    </section>
    

    <section class="cta" id="cta">
        <div class="cta-particles">
            <div class="particle"></div>
            <div class="particle"></div>
            <div class="particle"></div>
            <div class="particle"></div>
            <div class="particle"></div>
        </div>
                <div class="hero2-slider">
            <div class="hero2-slide active"></div>
            <div class="hero2-slide"></div>
            <div class="hero2-slide"></div>
            <div class="hero2-slide active"></div>
        </div>
        <div class="cta-container">
            <div class="cta-content bounce-in">
                <h2 class="cta-title">
                    <span class="title-word2">Ready</span>
                    <span class="title-word2">to</span>
                    <span class="title-word2">Join</span>
                    <span class="title-word2">the</span>
                    <span class="title-word2">Build Process?</span>
                </h2>
                <p class="cta3-description slide-left">
                    Site is in early stages of development, but feel free to join the discord and check out the map!
                </p>
                <div class="cta-buttons scale-up">
                    <a href="https://discord.gg/hug2T2u" class="btn btn-dark cta-btn-primary" target="_blank" rel="noopener noreferrer" aria-label="Join our Discord server to get started">
                        <i class="fab fa-discord" aria-hidden="true"></i>
                        <span>Join Discord</span>
                        <div class="btn-shine" aria-hidden="true"></div>
                    </a>
                    <a href="#team" class="btn btn-secondary btn-secondary" aria-label="Learn more about how to get started">
                        <i class="fas fa-info-circle" aria-hidden="true"></i>
                        <span>Meet The Team</span>
                        <div class="btn-shine" aria-hidden="true"></div>
                    </a>
                </div>
            </div>
        </div>
        
        <div class="cta-footer">
            <div class="cta-footer-inner">
                <p>© The Isle OCR Trackers - All rights reserved</p>
                <nav class="cta-footer-links" aria-label="Legal links">
                    <a href="" class="cta-footer-link-btn">Terms of Service</a>
                    <a href="" class="cta-footer-link-btn">Privacy Policy</a>
                </nav>
            </div>
        </div>
    </section>






    <button class="scroll-to-top" id="scrollToTop" aria-label="Scroll to top">
        <i class="fas fa-chevron-up" aria-hidden="true"></i>
    </button>


    <div class="gallery-modal" id="galleryModal">
        <div class="gallery-modal-content">
            <button class="gallery-modal-close" id="galleryModalClose" aria-label="Close image viewer">×</button>
            <button id="galleryModalPrev" class="gallery-modal-nav gallery-modal-prev" aria-label="Previous image">
                <i class="fas fa-chevron-left" aria-hidden="true"></i>
            </button>
            <button id="galleryModalNext" class="gallery-modal-nav gallery-modal-next" aria-label="Next image">
                <i class="fas fa-chevron-right" aria-hidden="true"></i>
            </button>
            <img id="galleryModalImage" src="" alt="">
        </div>
    </div>
    -->

    <script>
        if ('serviceWorker' in navigator) {
            window.addEventListener('load', function() {
                navigator.serviceWorker.register('/sw.js')
                    .then(function(registration) {
                        console.log('SW registered: ', registration);
                    })
                    .catch(function(registrationError) {
                        console.log('SW registration failed: ', registrationError);
                    });
            });
        }
    </script>
<script defer="" src="https://static.cloudflareinsights.com/beacon.min.js/v8c78df7c7c0f484497ecbca7046644da1771523124516" integrity="sha512-8DS7rgIrAmghBFwoOTujcf6D9rXvH8xm8JQ1Ja01h9QX8EzXldiszufYa4IFfKdLUKTTrnSFXLDkUEOTrZQ8Qg==" data-cf-beacon="{&quot;version&quot;:&quot;2024.11.0&quot;,&quot;token&quot;:&quot;480c3a4197e547738d78871ccf9b009f&quot;,&quot;r&quot;:1,&quot;server_timing&quot;:{&quot;name&quot;:{&quot;cfCacheStatus&quot;:true,&quot;cfEdge&quot;:true,&quot;cfExtPri&quot;:true,&quot;cfL4&quot;:true,&quot;cfOrigin&quot;:true,&quot;cfSpeedBrain&quot;:true},&quot;location_startswith&quot;:null}}" crossorigin="anonymous" style="display: none !important;"></script>


</body>
</html>
