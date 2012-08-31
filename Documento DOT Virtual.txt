Estágio de Desenvolvimento DOT Virtual - 31/08/2012

Alterações:
	-Permitir upload de mídia em qualquer formato
	-Incluir post a uma categoria padrao (DOT Virtual)
	-Criar método para inserir marca d'água
	-Na criação do post, adicionar termos de serviço vindo dinamicamente de uma página criada
	-Criação de canal do usuário, onde ele pode ver todos os seus posts e editá-los
	-Lista de tags pré-definidas no momento do upload para escolha do usuário

Pendências: 
	-Inserir marca d'água na exibição dos uploads.
	-Ajustar encoding para não corromper os arquivos quando tiverem caracteres especiais
	-Encontrar plugin que reproduz a mídia de acordo com seu formato
	-Limitar tamanho total ocupado em 300mb
	-Ajustar interface(CSS e JS)

----------------------------------------

Plugin Utilizado: Post From Site 
http://wordpress.org/extend/plugins/post-from-site/
http://redradar.net/category/plugins/post-from-site/

Alterações feitas:
Arquivo pfs-submit.php

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	Originalmente o plugin permitia que o upload fosse feito apenas com imagens, portanto, criamos um método que permite realizar o upload de qualquer mídia, ficando desta maneira:
	/*
	 * Method to upload any filetype.
	 * @author: Victor Kurauchi
	 * @since: 23/08/2012
	 */
	function upload_file($fileToUpload) {

		$file = wp_upload_bits( $fileToUpload["name"], null, file_get_contents($fileToUpload["tmp_name"]));

		if (false === $file['error']) {
			$wp_filetype = wp_check_filetype(basename($file['file']), null );
			$attachment = array(
					'post_mime_type' => $wp_filetype['type'],
					'post_title' => preg_replace('/\.[^.]+$/', '', basename($file['file'])),
					'post_content' => '',
					'post_status' => 'inherit',
					'post_parent' => $post->ID
			);
			$attach_id = wp_insert_attachment( $attachment, $file['file']);

			require_once(ABSPATH . "wp-admin" . '/includes/file.php');
			require_once(ABSPATH . 'wp-admin/includes/image.php');

			$attach_key = 'document_file_id';
			$attach_data = wp_generate_attachment_metadata( $attach_id, $file['file'] );

			wp_update_attachment_metadata( $attach_id,  $attach_data );

			return $attach_id;
		} else {
			//TODO: er, error handling?
			return false;
		}

	}
	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	Originalmente o plugin não permitia que o post inserido tivesse uma determinada categoria por padrão, portanto, foi feito um Hack para forçar que qualquer post inserido na sessão DOT Virtual seja da categoria DOT Virtual por padrão.
	
	Inserir shortcode na pagina desejada para DOT Virtual - [post-from-site cat='3']        // Onde cat é o ID da categoria que será indexada por padrão a todos os posts do DOT Virtual.
																							// Não esqueça de mudar o ID quando for criar a nova categoria DOTVirtual no novo Wordpress.
	
	->Na linha de código 26, onde: 
	$category = $pfs_data['cat']; 
	
	Nós obtemos a categoria que foi definida por padrão anteriormente no arquivo post-from-site.class.php
	no formulário de envio de mídia, criamos um input type 'Hidden', ficando desta maneira:
	
	No arquivo post-from-site.class.php
	// No momento de envio de formulário, criamos o input hidden para permitir a escolha da categoria pelo seu nome/id definida pela variável $cat.
	$out .= "<input type='hidden' name='cat' value='" .$cat. "' />\n";	
	
	------------

	No arquivo pfs-submit.php
	// Obtendo a categoria do input hidden:
	/* Forcar que todo post inserido seja indexado a categoria DOT Virtual*/
	$category = $pfs_data['cat'];
	
	E durante o momento de execução do método para inserir o post, associamos a categoria obtida ao post que será inserido.
	$postarr['post_category'] = array($category);						#Neste momento, o post é associado a categoria DOTVirtual
	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~  

	Originalmente o plugin não permitia que o upload realizado no momento de envio fosse indexado ao post criado.
	Exemplo: Ao inserir um post pelo DOT Virtual, após escolher a mídia que quer fazer upload, quando o formulário fosse submetido, não seria possível saber a qual post a mídia está relacionada.
	Para isto, foi alterado o código do formulário de envio, onde primeiro criamos o post, fazemos o upload da mídia e a indexamos ao post criado e depois fazemos um update no post para atualizar a mídia que foi inserida, 
	ficando desta maneira:
	
	// Criação do post
	$postarr = array();
	$postarr['post_title'] = $title;
	$postarr['comment_status'] = $pfs_options['comment_status'];
	$postarr['post_status'] = $pfs_options['post_status'];
	$postarr['post_author'] = ( is_user_logged_in() ) ? $user_ID : $pfs_options['default_author'];
	$postarr['tax_input'] = (array_key_exists('terms',$pfs_data)) ? $pfs_data['terms'] : array();
	$postarr['post_type'] = $pfs_options['post_type'];
	$postarr['post_category'] = array($category);						#Neste momento, o post é associado a categoria DOTVirtual
	$post_id = wp_insert_post($postarr);
	
	// Upload da mídia
	$file = $pfs_files['file-upload'];
	$result['file-upload'] = 'single';
	
	$fileToUpload = array( 'name'=>$file["name"][0], 'tmp_name'=>$file["tmp_name"][0] );
	$uploaded = wp_upload_bits( $fileToUpload["name"], null, file_get_contents($fileToUpload["tmp_name"]));
	
	if (false === $uploaded['error']) {
		$wp_filetype = wp_check_filetype(basename($uploaded['file']), null );
		$attachment = array(
				'post_mime_type' => $wp_filetype['type'],
				'post_title' => preg_replace('/\.[^.]+$/', '', basename($uploaded['file'])),
				'post_content' => '',
				'post_status' => 'inherit',
				'post_parent' => $post_id
		);
		
		$attach_id = wp_insert_attachment( $attachment, $uploaded['file'], $post_id);
	
		require_once(ABSPATH . "wp-admin" . '/includes/file.php');
		require_once(ABSPATH . 'wp-admin/includes/image.php');
	
		$attach_data = wp_generate_attachment_metadata( $attach_id, $uploaded['file'] );
	
		wp_update_attachment_metadata( $attach_id,  $attach_data );
	
		$upload[1] = $attach_id;
	} else {
		//TODO: er, error handling?
		return false;
	}
	
	// Atualizaçao do post para pertmir que o upload seja indexado corretamente.
	$my_post = array();
	$my_post['ID'] = $post_id;
	$my_post['post_content'] = $content . "<br/>"  . $post_id->post_content ;
	
	// Update the post into the database
	wp_update_post( $my_post );
	unset($my_post);
	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	Originalmente o plugin não tinha a parte 'Termos de Serviço' no momento do upload, portanto, criamos a parte dos termos vindo dinamicamente de uma página criada para facilitar alterações no conteúdo, ficando desta maneira: 
	
	// No arquivo post-from-site.class.php definimos uma variável para obter o ID da página desejada.
	$pageid = 86;             // Setado estaticamente para testes, em produção podemos obter pelo titulo get_page_by_title('Termos de serviço'); ou simplesmente por pageId.
	$pagedata = get_page( $pageid );
	
	// Já no formulário, incluímos algum código para permitir que o conteúdo deste campo de Termos venha de uma página criada anteriormente (page ID 86 - Termos) e o usuário não possa editar pelo front end.
	$out .= "<label for='terms-text'>". __('Termos de Serviço:','pfs_domain'). "</label><textarea id='terms-text' name='terms-text' rows='12' cols='50' readonly='readonly'> \n". $pagedata->post_content."</textarea>\n";
	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	Método criado para permitir a inclusao de marca d'água para ser exibida nas midias de upload.
	watermark.php
	<?php
	// Load the stamp and the photo to apply the watermark to
	$stamp = imagecreatefrompng('images/dotwatermark.png');
	$im = imagecreatefromjpeg('images/headers/trolley.jpg');

	// Set the margins for the stamp and get the height/width of the stamp image
	$marge_right = 10;
	$marge_bottom = 10;
	$sx = imagesx($stamp);
	$sy = imagesy($stamp);

	// Copy the stamp image onto our photo using the margin offsets and the photo 
	// width to calculate positioning of the stamp. 
	imagecopy($im, $stamp, imagesx($im) - $sx - $marge_right, imagesy($im) - $sy - $marge_bottom, 0, 0, imagesx($stamp), imagesy($stamp));

	// Output and free memory
	header('Content-type: image/png');
	imagepng($im);
	imagedestroy($im);
	?>
	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

	Página criada para o canal do usuário, já obtendo todos os seus posts e permitindo a edição do mesmo seguindo um template canal do usuário
	canal_usuario.php na raiz do tema.
	
	<?php 
		$pfs_options = $pfs_options_arr[0];
		
		if ( is_user_logged_in() ) {
			
			wp_get_current_user();
			$theauthorid = $current_user->ID;
			echo 'User ID: ' . $theauthorid . '<br />';
			echo 'Usu&aacute;rio: ' . $current_user->user_firstname . '<br />';
			echo 'Username: ' . $current_user->user_login . '<br />';
			
			// Query		
			query_posts( 'author=' . $theauthorid );
			
			?>
			<h1>Seus posts: </h1>
			
			<div id="post-byuser">
			<?php		
			if (!have_posts()) {
				?> <h2>Voce ainda n&atilde;o tem nenhum post! <a href="http://localhost/wdot/wordpress/?page_id=6">Clique aqui para criar um!</a></h2><?php
			} else { 			
			
				while ( have_posts() ) : the_post(); ?>			
					<div class="single">
					<?php  
					global $post;
					echo $post->ID;
					
					$files = array(
							'numberposts' => -1,
							'post_parent' => $post->ID, 
							'post_status' => 'inherit', 
							'post_type' => 'attachment', 
							'order' => 'ASC', 
							'orderby' => 'menu_order ID'
							 );
					$attachments = get_children($files);
					
					foreach ( $attachments as $attachment ) {
						//echo wp_get_attachment_link( $attachment->ID, 'thumbnail', true );
						echo wp_get_attachment_link($attachment->ID, 'medium');
					}
					
					?>
					<hr>
					</div>
				<?php endwhile; // end of the loop. 
				
				// Reset Query
				wp_reset_query();?>
				
				
				<?php 
			}
			?>
			
			</div>
			<!-- #post-byuser -->
			<h2 style="text-align: center"><a href="http://localhost/wdot/wordpress/?page_id=6">Fa&ccedil;a novos uploads</a></h2>	
			
			<?php 
		} else {
			echo "VOCE NAO TEM ACESSO";
		}
	
	?>
	
	##########################################################
	
Pendências:

	- Inserir marca d'água na exibição dos uploads.
	O método foi criado, porém será necessário fazer os ajustes para obter a imagem que foi postada pelo usuário e para cada imagem inserir a marca d'água. Assim, no momento que o usuário for visualizar ou baixar a mídia, 
	a marca d'água estará na imagem.
	
	~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	
	- Ajustar encoding para não corromper os arquivos quando tiverem caracteres especiais
	Pelo fato de diversos arquivos terem o seu nome com caracteres especiais, no momento em que o usuário for visualizar a página, ela nao será encontrada. Para isto, deverá ser feito uma alteração no momento de inserir o upload de mídia. Algo como a função sanitize ou urlencode no nome do arquivo quando for mandado para o banco de dados.
	Ou talvez mudar o encoding do banco do wordpress, e testar no Wordpress sem ser no servidor local.
	
	~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	
	- Encontrar plugin que reproduz a mídia de acordo com seu formato
	Atualmente quando o usuário clicar na mídia desejada, será direcionada para página de download do arquivo. A ideia é que haja algum reprodutor de mídia para todos os formatos, e seja reconhecida no momento em que o usuário
	clicar pra visualizar. 
	
	~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	
	- Limitar tamanho total ocupado em 300mb
	Configurar que o tamanho total dos arquivos enviados pro usuário tenha o limite de 300mb e o limite por arquivo seja de 50mb.
	Deve ser configurável via backend.
	
	~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	
	- Ajustar interface(CSS e JS)
	As classes, divs e ids já estão definidas, porém com a estilização padrão do plugin juntamente com o tema twenty eleven.
	Validar com javascript ou jquery o campo dos termos de serviço. Permitindo apenas o envio do formulário quando o usuário concordar com os termos	