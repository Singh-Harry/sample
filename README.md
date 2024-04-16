# PHP in the context of WordPress

## Custom plugin to deliver automatic updates for other custom plugins and themes via private GitHub releases.

Registering auto-loading of the classes – Since the plugin is quite simple and does not require any third-party dependencies, so instead of using composer based PSR-4 autoloading, I’m using a simple PHP function to autoload the classes.
```
/**
 * Autoloader
 *
 * @param string $class Class name passed to autoloader.
 *
 * @return void
 */
function githubUpdaterAutoloader($class)
{
	$namespace = 'GithubUpdater\\';

	if (strncmp($namespace, $class, strlen($namespace)) !== 0) {
    	    return;
	}

	$base_dir   = plugin_dir_path(__FILE__) . 'src' . DIRECTORY_SEPARATOR;
	$class	= substr($class, strlen($namespace));
	$file 	= $base_dir . str_replace('\\', DIRECTORY_SEPARATOR, $class) . '.php';

	if (file_exists($file)) {
    	    require_once $file;
	}
}
spl_autoload_register('githubUpdaterAutoloader');
```

Base class to handle update functionality
```
abstract class Base
{
    	protected $type;
    	protected $slug;
    	protected $version;
    	protected $cache;
    	protected $cache_key;
    	...

    	/**
     	* Base class constructor
     	*
     	* @param mixed $config Config passed as an array or a string.
     	*/
    	public function __construct($config)
    	{
        	if (! is_admin() || ! $config) {
            	return;
        	}

        	...
    	}

    	/**
     	* Base class init function to set required properties.
     	*
     	* @param mixed $config Config passed from __construct.
     	*
     	* @return void
     	*/
    	public function init( $config )
    	{
        	$this->cache = $config['cache'] ?? true;

        	...
    	}

    	/**
     	* Function to run the updater.
     	*
     	* @return void
     	*/
    	public function run()
    	{
        	add_filter('upgrader_pre_download', [$this, 'modifyHeaders'], 10, 1);
        	add_filter('upgrader_source_selection', [ $this, 'updateDirectoryName' ], 10, 4);
        	add_action('upgrader_process_complete', [$this, 'purge'], 10, 2);
    	}

    	/**
     	* Fetches repository data from Github.
     	*
     	* @return void
     	*/
    	public function request()
    	{
        	$remote = get_transient($this->cache_key);

        	if (false === $remote || ! $this->cache) {

            	$remote = wp_remote_get(
                	$this->api_url . '/releases/latest', [
                    	'timeout' => 10,
                    	'headers' => [
                        	'Accept' => 'application/json',
                        	'Authorization' => 'Bearer ' . $this->token
                    	]
                	]
            	);

            	if (is_wp_error($remote) || 200 !== wp_remote_retrieve_response_code($remote) || empty(wp_remote_retrieve_body($remote))) {
                	return false;
            	}

            	set_transient($this->cache_key, $remote, DAY_IN_SECONDS);

        	}

        	$remote = json_decode(wp_remote_retrieve_body($remote));

        	return $remote;
    	}

    	/**
     	* Updates transient object based if a new version is available.
     	*
     	* @param object $transient Wordpress transient object
     	*
     	* @return void
     	*/
    	public function update( $transient )
    	{
        	if (empty($transient->checked)) {
            	return $transient;
        	}

        	$remote = $this->request();

        	if ($remote && $this->compareVersions($this->version, $remote->tag_name, $this->minor)) {
            	$transient = $this->setResponse($transient, $remote);
        	}

        	return $transient;
    	}

    	/**
     	* Function to create and send the response back to update function.
     	*
     	* @param object $transient Wordpress transiet object
     	* @param object $remote	Remote data object
     	*
     	* @return object
     	*/
    	abstract public function setResponse($transient, $remote);

      ...
}
```

Plugin class extending the Base class to handle custom plugin updates
```
final class Plugin extends Base
{
    	protected $file;
    	protected $basename;
    	protected $plugin_data;

    	/**
     	* Plugin updater construct
     	*/
    	public function __construct($config)
    	{

        	if (is_array($config) && ! isset($config['file'])) {
            	return;
        	}

        	parent::__construct($config);

        	$this->init($config);
    	}

    	/**
     	* Plugin updater class init function to set required properties.
     	*
     	* @param mixed $config Config passed from __construct.
     	*
     	* @return void
     	*/
    	public function init($config)
    	{
        	$this->type = 'plugin';
        	$this->file = wp_normalize_path(is_array($config) ? $config['file'] : $config);
        	
        	...
    	}

    	/**
     	* Function to run the updater.
     	*
     	* @return void
     	*/
    	public function run()
    	{
        	add_filter('plugins_api', [ $this, 'info' ], 20, 3);
        	add_filter('site_transient_update_plugins', [ $this, 'update' ]);

        	parent::run();
    	}

    	/**
     	* Function to handle plugin information popup
     	*
     	* @param object $response Default response object.
     	* @param string $action   Wordpress current action.
     	* @param object $args 	Default arguments
     	*/
    	function info( $response, $action, $args )
    	{
        	if ('plugin' === $this->type && 'plugin_information' !== $action) {
            	return $response;
        	}

        	if (empty($args->slug) || $this->slug !== $args->slug) {
            	return $response;
        	}

        	$remote = $this->request();

        	if (! $remote) {
            	return $response;
        	}

        	$response = new \stdClass();

        	$response->name       	= $this->plugin_data['Name'];
        	$response->slug       	= $this->slug;
        	$response->version    	= $remote->tag_name;
        	
        	...
    	}

    	...
}
```

## Another custom plugin to handle payment gateway integration
Class constructor to initialize feature functions and return a singleton instance.
```
public static function instance() {

	if ( ! empty( self::$instance ) && ( self::$instance instanceof \<CLASS_NAME> ) ) {
    	  return self::$instance;
	}

	// Setup the singleton
	self::$instance = new <CLASS_NAME>();

	// Bootstrap
	self::$instance->setup_helpers();
	self::$instance->setup_gateways();

	if ( is_admin() ) {
    	  self::$instance->setup_admin();
	}

	add_action( 'rest_api_init', array( self::$instance, 'register_endpoints' ) );

	// Setup plugin tables on activation/upgrade.
	register_activation_hook( PLUGIN_FILE, array( self::$instance, 'setup_database_tables' ) );

	return self::$instance;

}
```

Registering custom REST API endpoints
```
public function register_endpoints() {

	$core_endpoints = array(
    	  '/<ENDPOINT>' => array(
        	'methods'         	=> \WP_REST_Server::READABLE,
        	'callback'        	=> array( $this, '<FUNCTION>' ),
        	'permission_callback'   => function() { return current_user_can('edit_posts'); },
    	  ),
	);

	$endpoints = array();

	$endpoints = array_merge( $core_endpoints, (array) apply_filters( '<ENDPOINTS_FILTER>', $endpoints ) );

	foreach ( $endpoints as $endpoint => $args ) {
    	  register_rest_route( '<NAMESPACE>/v1', $endpoint, $args );
	}
}
```

Adding dashboard settings page for a plugin
```
add_action( 'admin_menu', array( self::$instance, 'add_plugin_page' ) );
add_action( 'admin_init', array( self::$instance, 'add_page_settings' ) );
```
```
public function add_plugin_page() {

	add_menu_page(
    	  '<PAGE_TITLE>',
    	  '<MENU_TITLE>',
    	  'manage_options',
    	  '<SLUG>',
    	  null,
    	  'dashicons-money-alt,
    	  25
	);

	add_submenu_page(
    	  '<SLUG>',
    	  'Settings',
    	  'Settings',
    	  'manage_options',
    	  '<SLUG>',
    	  array( self::$instance, '<CALLBACK_FUNCTION>' ),
	);
}
```
```
public function add_page_settings() {
	register_setting(
    	  '<OPTION_GROUP>',
    	  '<OPTION_NAME>',
    	  array( 'sanitize_callback' => array( $this, '<SANITIZE_CALLBACK>' )
	);

	add_settings_section(
    	  '<SECTION_ID>',
    	  'Settings',
    	  array( $this, '<SECTION_CALLBACK>' ),
    	  '<SLUG>'
	);

	add_settings_field(
    	  '<SETTING_ID>',
    	  'API key',
    	  array( $this, '<SETTING_CALLBACK>' ),
    	  '<SLUG>',
    	  '<SECTION_ID>'
	);
}
```

# Gutenberg blocks

## Registering a custom entity to fetch products from a custom REST route
```
import { dispatch } from "@wordpress/data";

dispatch("core").addEntities([
  {
    name: "stripe/products",
    kind: "<CUSTOM_NAMESPACE>/v1",
    baseURL: "<CUSTOM_NAMESPACE>/v1/stripe/products",
  },
]);
```

Fetching products using the custom entity:
```
const { products, hasResolved, siteURL } = useSelect((select) => {
	const { getSite, getEntityRecord, hasFinishedResolution } =
  	select(coreStore);
	const args = ["<CUSTOM_NAMESPACE>/v1", "stripe/products"];
	const products = getEntityRecord(...args);
	const hasResolved = hasFinishedResolution("getEntityRecord", args);

	return {
  	siteURL: getSite()?.url,
  	products: products,
  	hasResolved: hasResolved,
	};
}, []);
```

## Adding a custom color settings field to a block.
1. Importing the required components
```
import {
  InspectorControls,
  useBlockProps,
  __experimentalColorGradientSettingsDropdown as ColorGradientSettingsDropdown,
  __experimentalUseMultipleOriginColorsAndGradients as useMultipleOriginColorsAndGradients,
} from "@wordpress/block-editor";
```
2. Adding additional color settings to the panel
```
<InspectorControls group="color">
  <ColorGradientSettingsDropdown
	__experimentalIsRenderedInSidebar
	settings={[
        {
            label: __("Table Text"),
            colorValue: tableTextColor,
            onColorChange: (newValue) => setAttributes({ tableTextColor: newValue }),
            isShownByDefault: true,
            resetAllFilter: () => ({
                tableTextColor: "var(--global-color-3)",
            }),
        },
        {
            colorValue: tableBackgroundColor,
            onColorChange: (newValue) =>
                setAttributes({ tableBackgroundColor: newValue }),
            label: __("Table Background"),
            resetAllFilter: () => ({
                tableBackgroundColor: "var(--global-color-4)",
            }),
        },
	]}
    panelId={clientId}
    {...colorGradientSettings}
  />
</InspectorControls>
```
3. Code snippet from a custom block
```
<div {...useBlockProps()}>
  <h3>{__('<BLOCK_NAME>','<DOMAIN>')}</h3>
  {!hasResolved && (
	<Placeholder>
  	<Spinner />
	</Placeholder>
  )}
  {hasResolved && products && (
	<div className="pricing">
  	{Object.keys(products).map((product) => {
    	if (
      	products[product].classes.includes("is-custom") &&
      	!showCustom
    	) {
      	return;
    	}
    	return (
      	<div
        	className={"plan " + products[product].classes.join(" ")}
        	style={{
          	color: tableTextColor,
          	backgroundColor: tableBackgroundColor,
        	}}
      	>
        	{products[product].image && (
          	<div className="image">
            	<img src={products[product].image} />
          	</div>
        	)}
        	<h2>{products[product].name}</h2>
        	<div className="description">
          	{products[product].description}
        	</div>
        	{products[product].features && (
          	<ul class="features">
            	{products[product].features.map((feature) => {
              	return (
                	<li>{feature}</li>
              	);
            	})}
          	</ul>
        	)}
      	</div>
    	);
  	})}
	</div>
  )}
</div>
```

## Overriding core block to add additional attributes
```
function addButtonOverrideSettings(settings, name) {
	if (name !== 'core/button') {
    	return settings;
	}

	settings.attributes = {
    	...settings.attributes,
    	buttonId: {
        	type: 'string',
        	source: 'attribute',
        	selector: 'a,button',
        	attribute: 'id',
    	},
	};

	return settings;
}

addFilter(
	'blocks.registerBlockType',
	'<NAMESPACE>/core-button/button-override-settings',
	addButtonOverrideSettings,
);
```
```
const addButtonOverrideInspectorControls = createHigherOrderComponent(
	(BlockEdit) => {
    	return (props) => {
        	const {
            	attributes: {
                	buttonId,
            	},
            	setAttributes,
            	name,
        	} = props;

        	if (name !== 'core/button') {
            	return <BlockEdit {...props} />;
        	}

        	return (
            	<>
                	<BlockEdit {...props} />
                	<InspectorControls>
                    	<PanelBody
                        	title={__('Additional settings', '<DOMAIN>')}
                        	initialOpen={true}
                    	>
                        	<TextControl
                            	label={__(Id attribute')}
                            	value={buttonId}
                            	onChange={(value) => setAttributes({ buttonId: value })}
                        	/>
                    	</PanelBody>
                	</InspectorControls>
            	</>
        	);
    	};
	},
	'withInspectorControl',
);

wp.hooks.addFilter(
	'editor.BlockEdit',
	'<NAMESPACE>/core-button/button-override-inspector-controls',
	addButtonOverrideInspectorControls,
);
```

# Javascript in the context of WordPress

Extending the default WordPress webpack config to enable support for multiple entry points
```
const defaultConfig = require("@wordpress/scripts/config/webpack.config");

const { getWebpackEntryPoints } = require("@wordpress/scripts/utils/config");

module.exports = {
  ...defaultConfig,
  entry: {
	...getWebpackEntryPoints(),
	"entry-1": "./src/entry-1.js",
	"entry-2": "./src/entry-2.js",
  },
};
```

Sending AJAX request to a custom WordPress REST route registered earlier
```
const response = await fetch(
    wpApiSettings.root + "<NAMESPACE>/v1/<ENDPOINT>",
    {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "X-WP-Nonce": wpApiSettings.nonce,
          },
          body: JSON.stringify({
            email: emailAddress,
            name: name,
            ...
          }),
    }
);
```

Some additional random Javascript snippets
```
async function buttonAction(event) {
  	event.preventDefault();
  	this.innerHTML = "Syncing";
  	this.disabled = true;

  	const action = this.dataset.action;
  	Const url = <URL>;

  	const response = await fetch(wpApiSettings.root + url, {
    	    headers: {
      	      "X-WP-Nonce": wpApiSettings.nonce,
    	    },
  	});

  	const { status } = await response.json();

  	if ("success" == status) {
    	    this.innerHTML = "Sync";
    	    this.disabled = false;
    	    updateGridData();
  	} else {
    	    ...
      }
}
```
```
document.addEventListener("DOMContentLoaded", function () {
  const toggles = document.querySelectorAll(".toggler input");
  const highlighter = document.querySelector(".toggler .highlighter");
  const priceOptions = document.querySelectorAll(
	".pricing .price"
  );

  toggles.forEach(function (toggle, index) {
	toggle.addEventListener("input", function (event) {
  	offset = (this.parentElement.clientWidth / toggles.length) * index - 12 * index || 0;
  	highlighter.style.setProperty("--offset",	this.nextElementSibling.offsetLeft + "px");
  	highlighter.style.setProperty("width", this.nextElementSibling.clientWidth + "px");

  	const activePrices = document.querySelectorAll(
    	  `.pricing .price.${this.id}, .pricing .button.${this.id}`
  	);
  	priceOptions.forEach((option) => option.classList.add("hide"));
  	activePrices.forEach((option) => option.classList.remove("hide"));
	});
});
```

# CI/CD - GitHub actions workflow config to automatically publish a release
```
name: Release WordPress theme

on:
  push:
	branches:
  	- release

permissions:
  contents: write

jobs:
  build-and-tag:
	runs-on: ubuntu-latest

	steps:
  	- uses: actions/checkout@v3

  	- name: Setup PHP
    	uses: shivammathur/setup-php@v2
    	with:
      	php-version: '8.3'
     	 
  	- name: Install Composer Dependencies
    	run: composer install --no-dev
   	 
  	- name: Setup Node
    	uses: actions/setup-node@v2
    	with:
      	node-version: '14'
     	 
  	- name: Install NPM Dependencies
    	run: npm install
   	 
  	- name: Build Theme
    	run: npm run build

  	- name: Extract theme version from style.css
    	
  	...
 
  	- name: Publish new release
    	env:
      	GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    	run: 
    	
  	...
```
