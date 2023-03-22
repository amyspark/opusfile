project(
	'opusfile',
	'c',
	version: '0.12',
	license: 'BSD-3-Clause',
	default_options: [
		'c_std=c89',
		'warning_level=3',
	],
	meson_version: '>=0.56',
)

project_arguments = []

compiler = meson.get_compiler('c')

is_windows = target_machine.system() == 'windows' or target_machine.system() == 'cygwin'

ogg = dependency('ogg', version: '>= 1.3', required: true)
opus = dependency('opus', version: '>= 1.0.1', required: true)
dependencies = [ogg, opus]

m = compiler.find_library('m',
	required: false,
)
if m.found()
	project_arguments += '-DOP_HAVE_LIBM'
	dependencies += m
endif
if compiler.has_header_symbol('math.h', 'lrintf', dependencies: m)
	project_arguments += '-DOP_HAVE_LRINTF'
endif

if compiler.get_argument_syntax() == 'msvc'
	add_project_arguments(
		compiler.get_supported_arguments(
			'-wd4267',
			'-wd4244',
			'-wd4090',
		),
		language: 'c',
	)
elif compiler.get_argument_syntax() == 'gcc'
	add_project_arguments(
		compiler.get_supported_arguments(
			'-Wno-parentheses',
			'-Wno-long-long',
		),
		language: 'c',
	)
endif

if not get_option('float_api')
	project_arguments += '-DOP_DISABLE_FLOAT_API'
endif

if not get_option('fixed_point')
	project_arguments += '-DOP_FIXED_POINT'
endif

if get_option('assertions')
	project_arguments += '-DOP_ENABLE_ASSERTIONS'
endif

opusfile_headers = files(
	'include/opusfile.h',
)

opusfile_srcs = files(
	'src/info.c',
	'src/internal.c',
	'src/opusfile.c',
  	'src/stream.c',
)

opusfile_includes = [
	include_directories('src'),
	include_directories('include'),
]

if is_windows
	opusfile = static_library(
		'opusfile',
		opusfile_srcs,
		include_directories: opusfile_includes,
		c_args: project_arguments,
		dependencies: dependencies,
		gnu_symbol_visibility: 'hidden',
		install: true,
	)
else
	opusfile = library(
		'opusfile',
		opusfile_srcs,
		include_directories: opusfile_includes,
		c_args: project_arguments,
		version: meson.project_version(),
		soversion: meson.project_version().split('.')[0],
		dependencies: dependencies,
		gnu_symbol_visibility: 'hidden',
		install: true,
	)
endif

install_headers(
	opusfile_headers,
	subdir: 'opus'
)

opusfile_dep = declare_dependency(
	link_with: opusfile,
	dependencies: dependencies,
	include_directories: opusfile_includes,
)

if get_option('http')
	opusurl_args = [
		'-DOP_ENABLE_HTTP',
	]

	openssl = dependency('openssl', required: true)

	opusurl_deps = [
		opusfile_dep,
		openssl,
	]

	if is_windows
		ws2_32 = compiler.find_library('ws2_32', required: true)
		crypt32 = compiler.find_library('crypt32', required: true)
		opusurl_deps += [
			ws2_32,
			crypt32,
		]
	else
		compiler.has_header('sys/socket.h', required: true)
	endif
	if compiler.compiles('''#include <time.h>
		int main(void)
		{
			struct timespec ts;
			return clock_gettime(CLOCK_REALTIME, &ts);
		}''',
		name: 'Supports clock_gettime'
	)
		opusurl_args += '-DOP_HAVE_CLOCK_GETTIME'
	elif compiler.has_header_symbol('sys/timeb.h', 'ftime')
		opusurl_args += '-DOP_HAVE_FTIME'
	else
		error('HTTP support requires either clock_gettime() or ftime()')
	endif

	opusurl_headers = files(
		'include/opusfile.h',
	)

	opusurl_srcs = files(
		'src/http.c',
		'src/internal.c',
	)

	if is_windows
		opusurl_srcs += files(
			'src/wincerts.c'
		)
	endif

	opusurl_includes = [
		include_directories('src'),
		include_directories('include'),
	]

	if is_windows
		opusurl = static_library(
			'opusurl',
			opusurl_srcs,
			include_directories: opusurl_includes,
			c_args: project_arguments + opusurl_args,
			dependencies: opusurl_deps,
			version: meson.project_version(),
			soversion: meson.project_version().split('.')[0],
		)
	else
		opusurl = static_library(
			'opusurl',
			opusurl_srcs,
			include_directories: opusurl_includes,
			c_args: project_arguments + opusurl_args,
			dependencies: opusurl_deps,
		)
	endif

	opusurl_dep = declare_dependency(
		link_with: opusurl,
		dependencies: opusurl_deps,
		include_directories: opusurl_includes
	)
else
	opusurl_dep = disabler()
endif

if get_option('examples')
	example_includes = include_directories('examples')

	opusfile_example_srcs = files(
		'examples/opusfile_example.c'
	)

	if is_windows
		opusfile_example_srcs += files(
			'examples/win32utf8.c',
		)
	endif

	opusfile_example = executable(
		'opusfile_example',
		opusfile_example_srcs,
		include_directories: example_includes,
		c_args: project_arguments,
		dependencies: [opusfile_dep, opusurl_dep,]
	)

	seeking_example_srcs = files(
		'examples/seeking_example.c'
	)

	if is_windows
		seeking_example_srcs += files(
			'examples/win32utf8.c',
		)
	endif

	seeking_example = executable(
		'seeking_example',
		seeking_example_srcs,
		include_directories: example_includes,
		c_args: project_arguments,
		dependencies: [opusfile_dep, opusurl_dep,]
	)
endif

if get_option('docs')
	doxygen = find_program('doxygen', required : false, disabler: true)
	dot = find_program('dot', required : false, disabler: true)

	cdata = configuration_data()
	cdata.set('PACKAGE_NAME', meson.project_name())
	cdata.set('PACKAGE_VERSION', meson.project_version())
	if dot.found()
		cdata.set('HAVE_DOT', 'YES')
	else
		cdata.set('HAVE_DOT', 'NO')
	endif
	cdata.set('top_srcdir', meson.current_source_dir())

	doxyfile = configure_file(
		input: 'doc/Doxyfile.in',
		output: 'Doxyfile',
		configuration: cdata,
	)

	dist_doc = get_option('datadir') / 'doc' / meson.project_name()

	docs = custom_target(
		'docs',
		input: doxyfile,
		output: 'html',
		command: [doxygen, doxyfile],
		console: true,
		install: true,
		install_dir: dist_doc,
	)

	install_data(
		files(
			'COPYING',
			'AUTHORS',
			'README.md',
		),
		install_dir: dist_doc
	)
endif

pkg = import('pkgconfig')

pkg.generate(
	opusfile,
	description: 'High-level Opus decoding library',
	subdirs: 'opus',
)

if get_option('examples')
	pkg.generate(
		opusurl,
		description: 'High-level Opus decoding library, URL support',
		subdirs: 'opus',
	)
endif
