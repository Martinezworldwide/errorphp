# errorphp

find the error in my code and explain why its not sliding my tab-content and displaying one at a time:

<?php
include 'db.php';
session_start();

if (isset($_GET['username'])) {
    $username = $_GET['username'];
    $stmt = $conn->prepare("SELECT id FROM users WHERE username = ?");
    $stmt->bind_param("s", $username);
    $stmt->execute();
    $stmt->bind_result($user_id);
    $stmt->fetch();
    $stmt->close();
} else {
    $user_id = $_SESSION['user_id'];
}
// ✅ Handle bio update
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_POST["bio"]) && isset($_POST['update_bio'])) {
    $new_bio = trim($_POST["bio"]);
    $stmt = $conn->prepare("UPDATE users SET bio=? WHERE id=?");
    $stmt->bind_param("si", $new_bio, $user_id);
    $stmt->execute();
    $stmt->close();
    header("Location: profile.php?username=" . urlencode($username));
    exit();
}

// ✅ Handle profile picture update
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["profile_picture"]["name"]) && isset($_POST['upload_profile_picture'])) {
    $target_dir = "uploads/";
    $file_name = time() . "_" . basename($_FILES["profile_picture"]["name"]);
    $target_file = $target_dir . $file_name;
    $imageFileType = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

    // Validate image
    if (getimagesize($_FILES["profile_picture"]["tmp_name"]) !== false) {
        // Limit file size to 2MB
        if ($_FILES["profile_picture"]["size"] <= 2097152) {
            // Allow only JPG, JPEG, PNG, GIF
            if (in_array($imageFileType, ["jpg", "jpeg", "png", "gif"])) {
                if (move_uploaded_file($_FILES["profile_picture"]["tmp_name"], $target_file)) {
                    // Update profile picture in DB
                    $stmt = $conn->prepare("UPDATE users SET profile_picture=? WHERE id=?");
                    $stmt->bind_param("si", $file_name, $user_id);
                    $stmt->execute();
                    $stmt->close();
                    header("Location: profile.php?username=" . urlencode($username));
                    exit();
                }
            }
        }
    }
}

// ✅ Ensure multiple image uploads work
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["gallery_images"]["name"]) && isset($_POST['upload_gallery_image'])) {
    $target_dir = "uploads/gallery/";

    // Ensure the folder exists
    if (!is_dir($target_dir)) {
        mkdir($target_dir, 0777, true);
    }

    foreach ($_FILES["gallery_images"]["tmp_name"] as $key => $tmp_name) {
        if (!empty($tmp_name)) {
            $file_name = time() . "_" . basename($_FILES["gallery_images"]["name"][$key]);
            $target_file = $target_dir . $file_name;
            $imageFileType = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

            // Validate image
            if (getimagesize($tmp_name) !== false) {
                // Limit file size to 5MB
                if ($_FILES["gallery_images"]["size"][$key] <= 5242880) {
                    // Allow only JPG, JPEG, PNG, GIF
                    if (in_array($imageFileType, ["jpg", "jpeg", "png", "gif"])) {
                        if (move_uploaded_file($tmp_name, $target_file)) {
                            // Save image path to database
                            $stmt = $conn->prepare("INSERT INTO user_images (user_id, image_path) VALUES (?, ?)");
                            $stmt->bind_param("is", $user_id, $target_file);
                            $stmt->execute();
                            $stmt->close();
                        }
                    }
                }
            }
        }
    }
    header("Location: profile.php?username=" . urlencode($username));
    exit();

    
}

// ✅ Handle Video Upload
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["video_file"]) && isset($_POST["upload_video"])) {
    $target_dir = "uploads/videos/";
    if (!is_dir($target_dir)) mkdir($target_dir, 0777, true);

    $file_name = time() . "_" . basename($_FILES["video_file"]["name"]);
    $target_file = $target_dir . $file_name;

    if (move_uploaded_file($_FILES["video_file"]["tmp_name"], $target_file)) {
        $stmt = $conn->prepare("INSERT INTO user_videos (user_id, video_path) VALUES (?, ?)");
        $stmt->bind_param("is", $user_id, $target_file);
        $stmt->execute();
        $stmt->close();
    }
}

// ✅ Handle Audio Upload
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["audio_file"]) && isset($_POST["upload_audio"])) {
    $target_dir = "uploads/audio/";
    if (!is_dir($target_dir)) mkdir($target_dir, 0777, true);

    $file_name = time() . "_" . basename($_FILES["audio_file"]["name"]);
    $target_file = $target_dir . $file_name;

    if (move_uploaded_file($_FILES["audio_file"]["tmp_name"], $target_file)) {
        $stmt = $conn->prepare("INSERT INTO user_audio (user_id, audio_path) VALUES (?, ?)");
        $stmt->bind_param("is", $user_id, $target_file);
        $stmt->execute();
        $stmt->close();
    }
}

// ✅ Handle Article Upload
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_POST["upload_article"])) {
    $title = trim($_POST["article_title"]);
    $content = trim($_POST["article_content"]);

    $stmt = $conn->prepare("INSERT INTO user_articles (user_id, title, content) VALUES (?, ?, ?)");
    $stmt->bind_param("iss", $user_id, $title, $content);
    $stmt->execute();
    $stmt->close();
}

// ✅ Handle Document Upload (PDF, DOCX, TXT, RT4)
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["document_file"]["name"]) && isset($_POST['upload_document'])) {
    $target_dir = "uploads/documents/";
    if (!is_dir($target_dir)) mkdir($target_dir, 0777, true);

    $file_name = time() . "_" . basename($_FILES["document_file"]["name"]);
    $target_file = $target_dir . $file_name;
    $file_type = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

    if (in_array($file_type, ["pdf", "docx", "txt", "rt4"])) {

        if (move_uploaded_file($_FILES["document_file"]["tmp_name"], $target_file)) {
            $stmt = $conn->prepare("INSERT INTO user_documents (user_id, title, file_path) VALUES (?, ?, ?)");
            $stmt->bind_param("iss", $user_id, $_FILES["document_file"]["name"], $target_file);
            $stmt->execute();
            $stmt->close();
        }
    }
    header("Location: profile.php?username=" . urlencode($username));
    exit();
}


// ✅ Handle General File Upload (Auto-Sorted by Type)
if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["gallery_files"]["name"]) && isset($_POST['upload_gallery_files'])) {
    $target_dir = "uploads/";
    if (!is_dir($target_dir)) mkdir($target_dir, 0777, true);

    foreach ($_FILES["gallery_files"]["tmp_name"] as $key => $tmp_name) {
        if (!empty($tmp_name)) {
            $file_name = time() . "_" . basename($_FILES["gallery_files"]["name"][$key]);
            $target_file = $target_dir . $file_name;
            $file_type = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

            // Determine where to store the file based on type
            if (in_array($file_type, ["jpg", "jpeg", "png", "gif"])) {
                $sub_dir = "images/";
            } elseif (in_array($file_type, ["mp4", "webm", "ogg"])) {
                $sub_dir = "videos/";
            } elseif (in_array($file_type, ["mp3", "wav", "ogg"])) {
                $sub_dir = "audio/";
            } elseif (in_array($file_type, ["txt", "pdf", "doc", "docx"])) {
                $sub_dir = "articles/";
            } else {
                continue; // Skip unsupported files
            }

            $final_path = $target_dir . $sub_dir . $file_name;
            if (!is_dir($target_dir . $sub_dir)) mkdir($target_dir . $sub_dir, 0777, true);

            if (move_uploaded_file($tmp_name, $final_path)) {
                // Insert into respective database tables
                if ($sub_dir == "images/") {
                    $stmt = $conn->prepare("INSERT INTO user_images (user_id, image_path) VALUES (?, ?)");
                } elseif ($sub_dir == "videos/") {
                    $stmt = $conn->prepare("INSERT INTO user_videos (user_id, video_path) VALUES (?, ?)");
                } elseif ($sub_dir == "audio/") {
                    $stmt = $conn->prepare("INSERT INTO user_audio (user_id, audio_path) VALUES (?, ?)");
                } elseif ($sub_dir == "articles/") {
                    $stmt = $conn->prepare("INSERT INTO user_articles (user_id, title, content) VALUES (?, ?, '')");
                    $file_name_clean = pathinfo($file_name, PATHINFO_FILENAME);
                    $stmt->bind_param("iss", $user_id, $file_name_clean, $final_path);
                }

                if ($sub_dir != "articles/") {
                    $stmt->bind_param("is", $user_id, $final_path);
                }
                $stmt->execute();
                $stmt->close();
            }
        }
    }
    header("Location: profile.php?username=" . urlencode($username));
    exit();
}


// Fetch user info
$stmt = $conn->prepare("SELECT username, bio, profile_picture FROM users WHERE id=?");
$stmt->bind_param("i", $user_id);
$stmt->execute();
$stmt->bind_result($username, $bio, $profile_picture);
$stmt->fetch();
$stmt->close();

// Fetch followers count
$stmt = $conn->prepare("SELECT COUNT(*) FROM followers WHERE following_id=?");
$stmt->bind_param("i", $user_id);
$stmt->execute();
$stmt->bind_result($followers_count);
$stmt->fetch();
$stmt->close();

// Fetch following count
$stmt = $conn->prepare("SELECT COUNT(*) FROM followers WHERE follower_id=?");
$stmt->bind_param("i", $user_id);
$stmt->execute();
$stmt->bind_result($following_count);
$stmt->fetch();
$stmt->close();

// Fetch followers list
$followers_query = "SELECT users.username FROM followers JOIN users ON followers.follower_id = users.id WHERE followers.following_id = ?";
$stmt = $conn->prepare($followers_query);
$stmt->bind_param("i", $user_id);
$stmt->execute();
$followers_result = $stmt->get_result();
$followers = [];
while ($row = $followers_result->fetch_assoc()) {
    $followers[] = $row['username'];
}
$stmt->close();

// Fetch following list
$following_query = "SELECT users.username FROM followers JOIN users ON followers.following_id = users.id WHERE followers.follower_id = ?";
$stmt = $conn->prepare($following_query);
$stmt->bind_param("i", $user_id);
$stmt->execute();
$following_result = $stmt->get_result();
$following = [];
while ($row = $following_result->fetch_assoc()) {
    $following[] = $row['username'];
}
$stmt->close();
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo htmlspecialchars($username); ?>'s Profile</title>
    <script src="js/jquery-3.6.0.min.js"></script>

</head>
<body>
    <h2>Welcome, <?php echo htmlspecialchars($username); ?>!</h2>
    <img src="uploads/<?php echo htmlspecialchars($profile_picture); ?>" width="100" alt="Profile Picture">
    
    <p><strong>Bio:</strong> <?php echo htmlspecialchars($bio) ?: "No bio yet"; ?></p>
    <?php if ($user_id == $_SESSION['user_id']) : ?>
<form method="POST">
    <label for="bio"><strong>✏️ Update Bio:</strong></label><br>
    <textarea name="bio" id="bio" rows="3" required><?php echo htmlspecialchars($bio); ?></textarea><br>
    <button type="submit" name="update_bio">Update</button>
</form>
<?php endif; ?>


    <h3>Followers (<?php echo $followers_count; ?>)</h3>
    <ul>
        <?php foreach ($followers as $follower): ?>
            <li><a href="profile.php?username=<?php echo urlencode($follower); ?>"><?php echo htmlspecialchars($follower); ?></a></li>
        <?php endforeach; ?>
    </ul>

    <h3>Following (<?php echo $following_count; ?>)</h3>
    <ul>
        <?php foreach ($following as $followed): ?>
            <li><a href="profile.php?username=<?php echo urlencode($followed); ?>"><?php echo htmlspecialchars($followed); ?></a></li>
        <?php endforeach; ?>
    </ul>

    <?php if ($user_id == $_SESSION['user_id']) : ?>
    
    <form method="POST" enctype="multipart/form-data">
    <label for="profile_picture"><strong>Change Profile Picture:</strong></label>
    <input type="file" name="profile_picture" id="profile_picture" required>
    <button type="submit" name="upload_profile_picture">Upload</button>
</form>
        <form method="POST" enctype="multipart/form-data">
    <label for="gallery_files"><strong>📂 Upload Files (Auto-Sorted):</strong></label>
    <input type="file" name="gallery_files[]" id="gallery_files" multiple required>
    <button type="submit" name="upload_gallery_files">Upload</button>
</form>


    <?php endif; ?>

    <!-- Profile Navigation Tabs -->
    <div>
    <button class="tab-button active" data-tab="tweets">Tweets</button>
    <button class="tab-button" data-tab="likes">Liked</button>
    <button class="tab-button" data-tab="images">Images</button>
    <button class="tab-button" data-tab="videos">Videos</button>
    <button class="tab-button" data-tab="music">Music & Sounds</button>
    <button class="tab-button" data-tab="articles">Articles</button>
</div>

   <!-- Tweets Section -->
<div id="tweets" class="tab-content active">
    <h3><?php echo htmlspecialchars($username); ?>'s Tweets</h3>
    <?php
    $tweets_query = "SELECT id, content, created_at FROM tweets WHERE user_id = ? ORDER BY created_at DESC";
    $stmt = $conn->prepare($tweets_query);
    $stmt->bind_param("i", $user_id);
    $stmt->execute();
    $tweets_result = $stmt->get_result();
    while ($tweet = $tweets_result->fetch_assoc()): ?>
        <div class="tweet-box">
            <p><?php echo htmlspecialchars($tweet['content']); ?></p>
            <small>(<?php echo $tweet['created_at']; ?>)</small>
        </div>
    <?php endwhile; ?>
</div>

<!-- Liked Tweets Section -->
<div id="likes" class="tab-content">
    <h3><?php echo htmlspecialchars($username); ?>'s Liked Tweets</h3>
    <?php
    $likes_query = "SELECT tweets.id, tweets.content, tweets.created_at, users.username 
                    FROM likes 
                    JOIN tweets ON likes.tweet_id = tweets.id 
                    JOIN users ON tweets.user_id = users.id 
                    WHERE likes.user_id = ? 
                    ORDER BY tweets.created_at DESC";
    $stmt = $conn->prepare($likes_query);
    $stmt->bind_param("i", $user_id);
    $stmt->execute();
    $likes_result = $stmt->get_result();
    while ($liked = $likes_result->fetch_assoc()): ?>
        <div class="liked-tweet-box">
            <p><strong><a href="profile.php?username=<?php echo urlencode($liked['username']); ?>">
                <?php echo htmlspecialchars($liked['username']); ?></a></strong>: 
            <?php echo htmlspecialchars($liked['content']); ?></p>
            <small>(<?php echo $liked['created_at']; ?>)</small>
        </div>
    <?php endwhile; ?>
</div>



   <!-- Images Section -->
<div id="images" class="tab-content">
    <h3>📸 <?php echo htmlspecialchars($username); ?>'s Gallery</h3>

    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>🖼 Upload Image</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="file" name="gallery_images[]" multiple required>
            <button type="submit" name="upload_gallery_image">Upload</button>
        </form>
    <?php endif; ?>

    <div class="gallery-grid">
        <?php
        $images_query = "SELECT image_path FROM user_images WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($images_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $images_result = $stmt->get_result();
        while ($image = $images_result->fetch_assoc()): ?>
            <div class="gallery-item">
                <a href="<?php echo htmlspecialchars($image['image_path']); ?>" target="_blank">
                    <img src="<?php echo htmlspecialchars($image['image_path']); ?>" alt="User Image">
                </a>
            </div>
        <?php endwhile; ?>
    </div>
</div>

<!-- Videos Section -->
<div id="videos" class="tab-content">
    <h3>🎥 <?php echo htmlspecialchars($username); ?>'s Videos</h3>

    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>📹 Upload Video</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="file" name="video_file" accept=".mp4,.webm,.ogg" required>
            <button type="submit" name="upload_video">Upload</button>
        </form>
    <?php endif; ?>

    <div class="gallery-grid">
        <?php
        $videos_query = "SELECT video_path FROM user_videos WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($videos_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $videos_result = $stmt->get_result();
        while ($video = $videos_result->fetch_assoc()): ?>
            <div class="gallery-item">
                <video width="250" controls>
                    <source src="<?php echo htmlspecialchars($video['video_path']); ?>" type="video/mp4">
                    Your browser does not support the video tag.
                </video>
            </div>
        <?php endwhile; ?>
    </div>
</div>
<!-- Music & Sounds Section -->
<div id="music" class="tab-content">
    <h3>🎵 <?php echo htmlspecialchars($username); ?>'s Music & Sounds</h3>

    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>🎧 Upload Audio</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="text" name="audio_title" placeholder="Enter song title" required>
            <input type="file" name="audio_file" accept="audio/mpeg, audio/ogg, audio/wav" required>
            <button type="submit" name="upload_audio">Upload</button>
        </form>
    <?php endif; ?>

    <div class="audio-grid">
        <?php
        $audio_query = "SELECT audio_path, title FROM user_audio WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($audio_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $audio_result = $stmt->get_result();
        while ($audio = $audio_result->fetch_assoc()): ?>
            <div class="audio-item">
                <div class="audio-title"><?php echo htmlspecialchars($audio['title']); ?></div>
                <audio controls>
                    <source src="<?php echo htmlspecialchars($audio['audio_path']); ?>" type="audio/mpeg">
                    Your browser does not support the audio element.
                </audio>
            </div>
        <?php endwhile; ?>
    </div>
</div>



    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>🎧 Upload Audio</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="file" name="audio_file" accept=".mp3,.wav,.ogg" required>

            <button type="submit" name="upload_audio">Upload</button>
        </form>
    <?php endif; ?>

    <div class="gallery-grid">
        <?php
        $audio_query = "SELECT audio_path FROM user_audio WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($audio_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $audio_result = $stmt->get_result();
        while ($audio = $audio_result->fetch_assoc()): ?>
            <div class="gallery-item">
                <audio controls>
                    <source src="<?php echo htmlspecialchars($audio['audio_path']); ?>" type="audio/mpeg">
                    Your browser does not support the audio element.
                </audio>
            </div>
        <?php endwhile; ?>
    </div>
</div>
<!-- Articles Section -->
<div id="articles" class="tab-content">
    <h3>📝 <?php echo htmlspecialchars($username); ?>'s Articles</h3>

    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>🖊 Write or Upload an Article</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="text" name="article_title" placeholder="Article Title" required><br>
            <textarea name="article_content" placeholder="Write your article here..." rows="6"></textarea><br>
            <button type="submit" name="upload_article">Publish</button>
        </form>

        <h4>📄 Upload a Document (PDF, DOCX, TXT)</h4>
        <form method="POST" enctype="multipart/form-data">
            <input type="file" name="document_file" accept=".pdf,.docx,.txt,.rt4" required>
            <button type="submit" name="upload_document">Upload</button>
        </form>
    <?php endif; ?>

    <div class="document-list">
        <?php
        $docs_query = "SELECT title, file_path FROM user_documents WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($docs_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $docs_result = $stmt->get_result();
        while ($doc = $docs_result->fetch_assoc()): ?>
            <div class="document-box">
                <h4><?php echo htmlspecialchars($doc['title']); ?></h4>
                <a href="<?php echo htmlspecialchars($doc['file_path']); ?>" target="_blank" class="document-link">📂 View Document</a>
            </div>
        <?php endwhile; ?>
    </div>
</div>




    <?php if ($user_id == $_SESSION['user_id']) : ?>
        <h4>🖊 Write an Article</h4>
        <form method="POST">
            <input type="text" name="article_title" placeholder="Article Title" required><br>
            <textarea name="article_content" placeholder="Write your article here..." rows="6" required></textarea><br>
            <button type="submit" name="upload_article">Publish</button>
        </form>
    <?php endif; ?>

    <div class="article-list">
        <?php
        $articles_query = "SELECT title, content, uploaded_at FROM user_articles WHERE user_id = ? ORDER BY uploaded_at DESC";
        $stmt = $conn->prepare($articles_query);
        $stmt->bind_param("i", $user_id);
        $stmt->execute();
        $articles_result = $stmt->get_result();
        while ($article = $articles_result->fetch_assoc()): ?>
            <div class="article-box">
                <h4><?php echo htmlspecialchars($article['title']); ?></h4>
                <p><?php echo nl2br(htmlspecialchars($article['content'])); ?></p>
                <small>(<?php echo $article['uploaded_at']; ?>)</small>
            </div>
        <?php endwhile; ?>
    </div>
</div>


    <div style="margin-top: 20px;">
        <a href="home.php">🏠 Back to Home</a> | <a href="logout.php">🚪 Logout</a>
    </div>

  <script>
    $(document).ready(function() {
        // Hide all tabs except the first one
        $(".tab-content").hide();
        $(".tab-content:first").show();

        // Handle tab clicks
        $(".tab-button").click(function() {
            $(".tab-button").removeClass("active");
            $(this).addClass("active");

            let target = $(this).attr("data-tab");
            $(".tab-content").hide();
            $("#" + target).fadeIn(); // Add fade-in effect
        });
    });
</script>


    <script>
    $(document).ready(function() {
        alert("jQuery is working!");
    });
</script>


    <style>
.tab-content {
    display: none; /* Hide all tabs by default */
}
.tab-content:first-of-type {
    display: block; /* Show first tab on page load */
}
.tab-button.active {
    background-color: #ddd; /* Highlight active tab */
}


/* Style for active tab button */
.tab-button.active {
    background-color: #007bff; /* Change to match your theme */
    color: white;
    border: none;
}

/* General tab button styling */
.tab-button {
    padding: 10px 15px;
    border: 1px solid #ccc;
    background: #f9f9f9;
    cursor: pointer;
    transition: 0.3s;
}

.tab-button:hover {
    background: #ddd;
}

/* Tweet & Liked Tweet Styling */
.tweet-box, .liked-tweet-box {
    border: 1px solid #ccc;
    padding: 10px;
    margin: 10px auto;
    width: 90%;
    max-width: 600px;
    text-align: left;
    background: #f9f9f9;
    border-radius: 8px;
}

/* Gallery Layout */
.gallery-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    gap: 10px;
    padding: 10px;
    justify-content: center;
}

.gallery-item img {
    width: 100%;
    height: auto;
    object-fit: cover;
    border-radius: 8px;
    cursor: pointer;
}

.gallery-item img:hover {
    opacity: 0.8;
}

/* SoundCloud-style audio display */
.audio-grid {
    display: flex;
    flex-direction: column;
    gap: 15px;
    padding: 15px;
    justify-content: center;
    max-width: 800px;
    margin: auto;
}

.audio-item {
    background: #f3f3f3;
    padding: 10px;
    border-radius: 10px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    text-align: center;
}

.audio-title {
    font-weight: bold;
    font-size: 16px;
    margin-bottom: 5px;
    color: #333;
}

/* Document Styling */
.document-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
    max-width: 600px;
    margin: auto;
}

.document-box {
    background: #f4f4f4;
    padding: 10px;
    border-radius: 8px;
    text-align: center;
}

.document-link {
    text-decoration: none;
    font-weight: bold;
    color: #007bff;
}

.document-link:hover {
    text-decoration: underline;
}

</style>


</body>
</html>
