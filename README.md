<?php
/*
Plugin Name: WP Silici
Description: Yorumları, yazıları ve sayfaları topluca silmek için basit ve hızlı bir eklenti. Silinecek içerik sayısını gösterir, onaylanmamış yorumlar için de seçenek sunar.
Version: 1.2
Author: Mustafa Çakar
*/

// Admin menüye sayfa ekleme
add_action('admin_menu', function() {
    add_menu_page(
        'WP Silici',
        'WP Silici',
        'manage_options',
        'wp-silici',
        'wp_silici_admin_page',
        'dashicons-trash',
        80
    );
});

// Admin sayfası içeriği
function wp_silici_admin_page() {
    if (!current_user_can('manage_options')) {
        return;
    }

    global $wpdb;

    // Yazı ve sayfa sayıları
    $post_count = wp_count_posts('post')->publish + wp_count_posts('post')->draft + wp_count_posts('post')->pending + wp_count_posts('post')->trash + wp_count_posts('post')->future + wp_count_posts('post')->private;
    $page_count = wp_count_posts('page')->publish + wp_count_posts('page')->draft + wp_count_posts('page')->pending + wp_count_posts('page')->trash + wp_count_posts('page')->future + wp_count_posts('page')->private;

    // Yorum sayıları
    $comment_count_all = $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->comments}");
    // Onaylanmamış yorumlar: comment_approved != '1' (0=beklemede, spam, çöp)
    $comment_count_unapproved = $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->comments} WHERE comment_approved != '1'");

    // Silme işlemi
    if (isset($_POST['wp_silici_action']) && check_admin_referer('wp_silici_bulk_delete_action', 'wp_silici_nonce_field')) {
        $deleted_counts = [];

        if (!empty($_POST['delete_posts'])) {
            $posts = get_posts([
                'numberposts' => -1,
                'post_type' => 'post',
                'post_status' => 'any',
                'fields' => 'ids',
            ]);
            foreach ($posts as $post_id) {
                wp_delete_post($post_id, true);
            }
            $deleted_counts['posts'] = count($posts);
            $post_count = 0;
        }

        if (!empty($_POST['delete_pages'])) {
            $pages = get_posts([
                'numberposts' => -1,
                'post_type' => 'page',
                'post_status' => 'any',
                'fields' => 'ids',
            ]);
            foreach ($pages as $page_id) {
                wp_delete_post($page_id, true);
            }
            $deleted_counts['pages'] = count($pages);
            $page_count = 0;
        }

        if (!empty($_POST['delete_comments_all'])) {
            $wpdb->query("DELETE FROM {$wpdb->comments}");
            $wpdb->query("DELETE FROM {$wpdb->commentmeta}");
            $deleted_counts['comments_all'] = true;
            $comment_count_all = 0;
            $comment_count_unapproved = 0;
        }

        if (!empty($_POST['delete_comments_unapproved'])) {
            // Sadece onaylanmamış yorumları sil
            $wpdb->query("DELETE FROM {$wpdb->comments} WHERE comment_approved != '1'");
            // Onlara ait meta kayıtları da silelim
            $wpdb->query("DELETE cmeta FROM {$wpdb->commentmeta} cmeta LEFT JOIN {$wpdb->comments} c ON cmeta.comment_id = c.comment_ID WHERE c.comment_ID IS NULL");
            $deleted_counts['comments_unapproved'] = true;
            $comment_count_all -= $comment_count_unapproved;
            $comment_count_unapproved = 0;
        }

        echo '<div class="notice notice-success is-dismissible"><p>';
        echo 'Silme işlemi tamamlandı. ';
        if (isset($deleted_counts['posts'])) echo $deleted_counts['posts'] . ' yazı silindi. ';
        if (isset($deleted_counts['pages'])) echo $deleted_counts['pages'] . ' sayfa silindi. ';
        if (isset($deleted_counts['comments_all'])) echo 'Tüm yorumlar silindi. ';
        if (isset($deleted_counts['comments_unapproved'])) echo 'Onaylanmamış yorumlar silindi.';
        echo '</p></div>';
    }

    ?>
    <div class="wrap">
        <h1>WP Silici - Toplu Silme Aracı</h1>
        <form method="post">
            <?php wp_nonce_field('wp_silici_bulk_delete_action', 'wp_silici_nonce_field'); ?>
            <table class="form-table">
                <tr>
                    <th scope="row">Yazıları sil</th>
                    <td>
                        <label><input type="checkbox" name="delete_posts" value="1"> Seçildi (Toplam: <strong><?php echo $post_count; ?></strong>)</label>
                    </td>
                </tr>
                <tr>
                    <th scope="row">Sayfaları sil</th>
                    <td>
                        <label><input type="checkbox" name="delete_pages" value="1"> Seçildi (Toplam: <strong><?php echo $page_count; ?></strong>)</label>
                    </td>
                </tr>
                <tr>
                    <th scope="row">Tüm yorumları sil</th>
                    <td>
                        <label><input type="checkbox" name="delete_comments_all" value="1"> Seçildi (Toplam: <strong><?php echo $comment_count_all; ?></strong>)</label>
                    </td>
                </tr>
                <tr>
                    <th scope="row">Onaylanmamış yorumları sil</th>
                    <td>
                        <label><input type="checkbox" name="delete_comments_unapproved" value="1"> Seçildi (Toplam: <strong><?php echo $comment_count_unapproved; ?></strong>)</label>
                    </td>
                </tr>
            </table>
            <p><em>Not: "Tüm yorumları sil" seçeneği işaretlenirse, "Onaylanmamış yorumları sil" seçeneği otomatik etkisiz kalmalıdır.</em></p>
            <p class="submit">
                <input type="submit" name="wp_silici_action" class="button button-primary" value="Seçilenleri Sil">
            </p>
        </form>

        <script>
            // JS ile iki yorum seçeneğini uyumlu yapalım (isteğe bağlı)
            document.addEventListener('DOMContentLoaded', function () {
                const allComments = document.querySelector('input[name="delete_comments_all"]');
                const unapprovedComments = document.querySelector('input[name="delete_comments_unapproved"]');

                function toggleUnapproved() {
                    if (allComments.checked) {
                        unapprovedComments.checked = false;
                        unapprovedComments.disabled = true;
                    } else {
                        unapprovedComments.disabled = false;
                    }
                }

                allComments.addEventListener('change', toggleUnapproved);
                toggleUnapproved();
            });
        </script>
    </div>
    <?php
}
