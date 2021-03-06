[![Build Status](https://travis-ci.org/FormFu/Catalyst-Controller-HTML-FormFu.svg?branch=master)](https://travis-ci.org/FormFu/Catalyst-Controller-HTML-FormFu)
[![Kwalitee status](http://cpants.cpanauthors.org/dist/Catalyst-Controller-HTML-FormFu.png)](http://cpants.charsbar.org/dist/overview/Catalyst-Controller-HTML-FormFu)
[![GitHub issues](https://img.shields.io/github/issues/FormFu/Catalyst-Controller-HTML-FormFu.svg)](https://github.com/FormFu/Catalyst-Controller-HTML-FormFu/issues)
[![GitHub tag](https://img.shields.io/github/tag/FormFu/Catalyst-Controller-HTML-FormFu.svg)]()
[![Cpan license](https://img.shields.io/cpan/l/Catalyst-Controller-HTML-FormFu.svg)](https://metacpan.org/release/Catalyst-Controller-HTML-FormFu)
[![Cpan version](https://img.shields.io/cpan/v/Catalyst-Controller-HTML-FormFu.svg)](https://metacpan.org/release/Catalyst-Controller-HTML-FormFu)

# NAME

Catalyst::Controller::HTML::FormFu - Catalyst integration for HTML::FormFu

# VERSION

version 2.04

# SYNOPSIS

    package MyApp::Controller::My::Controller;

    use Moose;
    use namespace::autoclean;

    BEGIN { extends 'Catalyst::Controller::HTML::FormFu'; }

    sub index : Local {
        my ( $self, $c ) = @_;

        # doesn't use an Attribute to make a form
        # can get an empty form from $self->form()

        my $form = $self->form();
    }

    sub foo : Local : Form {
        my ( $self, $c ) = @_;

        # using the Form attribute is equivalent to:
        #
        # my $form = $self->form;
        #
        # $form->process;
        #
        # $c->stash->{form} = $form;
    }

    sub bar : Local : FormConfig {
        my ( $self, $c ) = @_;

        # using the FormConfig attribute is equivalent to:
        #
        # my $form = $self->form;
        #
        # $form->load_config_filestem('root/forms/my/controller/bar');
        #
        # $form->process;
        #
        # $c->stash->{form} = $form;
        #
        # so you only need to do the following...

        my $form = $c->stash->{form};

        if ( $form->submitted_and_valid ) {
            do_something();
        }
    }

    sub baz : Local : FormConfig('my_config') {
        my ( $self, $c ) = @_;

        # using the FormConfig attribute with an argument is equivalent to:
        #
        # my $form = $self->form;
        #
        # $form->load_config_filestem('root/forms/my_config');
        #
        # $form->process;
        #
        # $c->stash->{form} = $form;
        #
        # so you only need to do the following...

        my $form = $c->stash->{form};

        if ( $form->submitted_and_valid ) {
            do_something();
        }
    }

    sub quux : Local : FormMethod('load_form') {
        my ( $self, $c ) = @_;

        # using the FormMethod attribute with an argument is equivalent to:
        #
        # my $form = $self->form;
        #
        # $form->populate( $c->load_form );
        #
        # $form->process;
        #
        # $c->stash->{form} = $form;
        #
        # so you only need to do the following...

        my $form = $c->stash->{form};

        if ( $form->submitted_and_valid ) {
            do_something();
        }
    }

    sub load_form {
        my ( $self, $c ) = @_;

        # Automatically called by the above FormMethod('load_form') action.
        # Called as a method on the controller object, with the context
        # object as an argument.

        # Must return a hash-ref suitable to be fed to $form->populate()
    }

You can also use specially-named actions that will only be called under certain
circumstances.

    sub edit : Chained('group') : PathPart : Args(0) : FormConfig { }

    sub edit_FORM_VALID {
        my ( $self, $c ) = @_;

        my $form  = $c->stash->{form};
        my $group = $c->stash->{group};

        $form->model->update( $group );

        $c->response->redirect( $c->uri_for( '/group', $group->id ) );
    }

    sub edit_FORM_NOT_SUBMITTED {
        my ( $self, $c ) = @_;

        my $form  = $c->stash->{form};
        my $group = $c->stash->{group};

        $form->model->default_values( $group );
    }

# METHODS

## form

This creates a new [HTML::FormFu](https://metacpan.org/pod/HTML::FormFu) object, passing as it's argument the
contents of the ["constructor"](#constructor) config value.

This is useful when using the ConfigForm() or MethodForm() action attributes,
to create a 2nd form which isn't populated using a config-file or method return
value.

    sub foo : Local {
        my ( $self, $c ) = @_;

        my $form = $self->form;
    }

Note that when using this method, the form's [query](https://metacpan.org/pod/HTML::FormFu#query) method
is not populated with the Catalyst request object.

# SPECIAL ACTION NAMES

An example showing how a complicated action method can be broken down into
smaller sections, making it clearer which code will be run, and when.

    sub edit : Local : FormConfig {
        my ( $self, $c ) = @_;

        my $form  = $c->stash->{form};
        my $group = $c->stash->{group};

        $c->detach('/unauthorised') unless $c->user->can_edit( $group );

        if ( $form->submitted_and_valid ) {
            $form->model->update( $group );

            $c->response->redirect( $c->uri_for('/group', $group->id ) );
            return;
        }
        elsif ( !$form->submitted ) {
            $form->model->default_values( $group );
        }

        $self->_add_breadcrumbs_nav( $c, $group );
    }

Instead becomes...

    sub edit : Local : FormConfig {
        my ( $self, $c ) = @_;

        $c->detach('/unauthorised') unless $c->user->can_edit(
            $c->stash->{group}
        );
    }

    sub edit_FORM_VALID {
        my ( $self, $c ) = @_;

        my $group = $c->stash->{group};

        $c->stash->{form}->model->update( $group );

        $c->response->redirect( $c->uri_for('/group', $group->id ) );
    }

    sub edit_FORM_NOT_SUBMITTED {
        my ( $self, $c ) = @_;

        $c->stash->{form}->model->default_values(
            $c->stash->{group}
        );
    }

    sub edit_FORM_RENDER {
        my ( $self, $c ) = @_;

        $self->_add_breadcrumbs_nav( $c, $c->stash->{group} );
    }

For any action method that uses a `Form`, `FormConfig` or `FormMethod`
attribute, you can add extra methods that use the naming conventions below.

These methods will be called after the original, plainly named action method.

## \_FORM\_VALID

Run when the form has been submitted and has no errors.

## \_FORM\_SUBMITTED

Run when the form has been submitted, regardless of whether or not there was
errors.

## \_FORM\_COMPLETE

For MultiForms, is run if the MultiForm is completed.

## \_FORM\_NOT\_VALID

Run when the form has been submitted and there were errors.

## \_FORM\_NOT\_SUBMITTED

Run when the form has not been submitted.

## \_FORM\_NOT\_COMPLETE

For MultiForms, is run if the MultiForm is not completed.

## \_FORM\_RENDER

For normal `Form` base classes, this subroutine is run after any of the other
special methods, unless `$form->submitted_and_valid` is true.

For `MultiForm` base classes, this subroutine is run after any of the other
special methods, unless `$multi->complete` is true.

# CUSTOMIZATION

You can set your own config settings, using either your controller config or
your application config.

    $c->config( 'Controller::HTML::FormFu' => \%my_values );

    # or

    MyApp->config( 'Controller::HTML::FormFu' => \%my_values );

    # or, in myapp.conf

    <Controller::HTML::FormFu>
        default_action_use_path 1
    </Controller::HTML::FormFu>

## form\_method

Override the method-name used to create a new form object.

See ["form"](#form).

Default value: `form`.

## form\_stash

Sets the stash key name used to store the form object.

Default value: `form`.

## form\_attr

Sets the attribute name used to load the
[Catalyst::Controller::HTML::FormFu::Action::Form](https://metacpan.org/pod/Catalyst::Controller::HTML::FormFu::Action::Form) action.

Default value: `Form`.

## config\_attr

Sets the attribute name used to load the
[Catalyst::Controller::HTML::FormFu::Action::Config](https://metacpan.org/pod/Catalyst::Controller::HTML::FormFu::Action::Config) action.

Default value: `FormConfig`.

## method\_attr

Sets the attribute name used to load the
[Catalyst::Controller::HTML::FormFu::Action::Method](https://metacpan.org/pod/Catalyst::Controller::HTML::FormFu::Action::Method) action.

Default value: `FormMethod`.

## form\_action

Sets which package will be used by the Form() action.

Probably only useful if you want to create a sub-class which provides custom
behaviour.

Default value: `Catalyst::Controller::HTML::FormFu::Action::Form`.

## config\_action

Sets which package will be used by the Config() action.

Probably only useful if you want to create a sub-class which provides custom
behaviour.

Default value: `Catalyst::Controller::HTML::FormFu::Action::Config`.

## method\_action

Sets which package will be used by the Method() action.

Probably only useful if you want to create a sub-class which provides custom
behaviour.

Default value: `Catalyst::Controller::HTML::FormFu::Action::Method`.

## constructor

Pass common defaults to the [HTML::FormFu constructor](https://metacpan.org/pod/HTML::FormFu#new).

These values are used by all of the action attributes, and by the `$self->form` method.

Default value: `{}`.

## config\_callback

Arguments: bool

If true, a coderef is passed to `$form->config_callback->{plain_value}`
which replaces any instance of `__uri_for(URI)__` found in form config files
with the result of passing the `URI` argument to ["uri\_for" in Catalyst](https://metacpan.org/pod/Catalyst#uri_for).

The form `__uri_for(URI, PATH, PARTS)__` is also supported, which is
equivalent to `$c->uri_for( 'URI', \@ARGS )`. At this time, there is no
way to pass query values equivalent to `$c->uri_for( 'URI', \@ARGS,
\%QUERY_VALUES )`.

The second codeword that is being replaced is `__path_to( @DIRS )__`. Any
instance is replaced with the result of passing the `DIRS` arguments to
["path\_to" in Catalyst](https://metacpan.org/pod/Catalyst#path_to). Don't use qoutationmarks as they would become part of the
path.

Default value: 1

## default\_action\_use\_name

If set to a true value the action for the form will be set to the currently
called action name.

Default value: `false`.

## default\_action\_use\_path

If set to a true value the action for the form will be set to the currently
called action path. The action path includes concurrent to action name
additioal parameters which were code inside the path.

Default value: `false`.

Example:

    action: /foo/bar
    called uri contains: /foo/bar/1

    # default_action_use_name => 1 leads to:
    $form->action = /foo/bar

    # default_action_use_path => 1 leads to:
    $form->action = /foo/bar/1

## model\_stash

Arguments: \\%stash\_keys\_to\_model\_names

Used to place Catalyst models on the form stash.

If it's being used to make a [DBIx::Class](https://metacpan.org/pod/DBIx::Class) schema available for
["options\_from\_model" in HTML::FormFu::Model::DBIC](https://metacpan.org/pod/HTML::FormFu::Model::DBIC#options_from_model), for `Select` and other
Group-type elements - then the hash-key must be `schema`. For example, if your
schema model class is `MyApp::Model::MySchema`, you would set `model_stash`
like so:

    <Controller::HTML::FormFu>
        <model_stash>
            schema MySchema
        </model_stash>
    </Controller::HTML::FormFu>

## context\_stash

To allow your form validation packages, etc, access to the catalyst context, a
weakened reference of the context is copied into the form's stash.

    $form->stash->{context};

This setting allows you to change the key name used in the form stash.

Default value: `context`

## languages\_from\_context

If you're using a L10N / I18N plugin such as [Catalyst::Plugin::I18N](https://metacpan.org/pod/Catalyst::Plugin::I18N) which
provides a `languages` method that returns a list of valid languages to use
for the currect request - and you want to use formfu's built-in I18N packages,
then setting ["languages\_from\_context"](#languages_from_context)

## localize\_from\_context

If you're using a L10N / I18N plugin such as [Catalyst::Plugin::I18N](https://metacpan.org/pod/Catalyst::Plugin::I18N) which
provides it's own `localize` method, you can set [localize\_from\_context](https://metacpan.org/pod/localize_from_context) to
use that method for formfu's localization.

## request\_token\_enable

If true, adds an instance of [HTML::FormFu::Plugin::RequestToken](https://metacpan.org/pod/HTML::FormFu::Plugin::RequestToken) to every
form, to stop accidental double-submissions of data and to prevent CSRF
attacks.

## request\_token\_field\_name

Defaults to `_token`.

## request\_token\_session\_key

Defaults to `__token`.

## request\_token\_expiration\_time

Defaults to `3600`.

# DISCONTINUED CONFIG SETTINGS

## config\_file\_ext

Support for this has now been removed. Config files are now searched for, with
any file extension supported by Config::Any.

## config\_file\_path

Support for this has now been removed. Use `{constructor}{config_file_path}` instead.

# CAVEATS

When using the `Form` action attribute to create an empty form, you must call
[$form->process](https://metacpan.org/pod/HTML::FormFu#process) after populating the form. However,
you don't need to pass any arguments to `process`, as the Catalyst request
object will have automatically been set in [$form->query](https://metacpan.org/pod/HTML::FormFu#query).

When using the `FormConfig` and `FormMethod` action attributes, if you make
any modifications to the form, such as adding or changing it's elements, you
must call [$form->process](https://metacpan.org/pod/HTML::FormFu#process) before rendering the form.

# AUTHORS

- Carl Franks <cpan@fireartist.com>
- Nigel Metheringham <nigelm@cpan.org>
- Dean Hamstead <dean@bytefoundry.com.au>

# COPYRIGHT AND LICENSE

This software is copyright (c) 2007-2018 by Carl Franks / Nigel Metheringham / Dean Hamstead.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.

# SUPPORT

## Perldoc

You can find documentation for this module with the perldoc command.

    perldoc Catalyst::Controller::HTML::FormFu

## Websites

The following websites have more information about this module, and may be of help to you. As always,
in addition to those websites please use your favorite search engine to discover more resources.

- MetaCPAN

    A modern, open-source CPAN search engine, useful to view POD in HTML format.

    [http://metacpan.org/release/Catalyst-Controller-HTML-FormFu](http://metacpan.org/release/Catalyst-Controller-HTML-FormFu)

- Search CPAN

    The default CPAN search engine, useful to view POD in HTML format.

    [http://search.cpan.org/dist/Catalyst-Controller-HTML-FormFu](http://search.cpan.org/dist/Catalyst-Controller-HTML-FormFu)

- RT: CPAN's Bug Tracker

    The RT ( Request Tracker ) website is the default bug/issue tracking system for CPAN.

    [https://rt.cpan.org/Public/Dist/Display.html?Name=Catalyst-Controller-HTML-FormFu](https://rt.cpan.org/Public/Dist/Display.html?Name=Catalyst-Controller-HTML-FormFu)

- AnnoCPAN

    The AnnoCPAN is a website that allows community annotations of Perl module documentation.

    [http://annocpan.org/dist/Catalyst-Controller-HTML-FormFu](http://annocpan.org/dist/Catalyst-Controller-HTML-FormFu)

- CPAN Ratings

    The CPAN Ratings is a website that allows community ratings and reviews of Perl modules.

    [http://cpanratings.perl.org/d/Catalyst-Controller-HTML-FormFu](http://cpanratings.perl.org/d/Catalyst-Controller-HTML-FormFu)

- CPAN Forum

    The CPAN Forum is a web forum for discussing Perl modules.

    [http://cpanforum.com/dist/Catalyst-Controller-HTML-FormFu](http://cpanforum.com/dist/Catalyst-Controller-HTML-FormFu)

- CPANTS

    The CPANTS is a website that analyzes the Kwalitee ( code metrics ) of a distribution.

    [http://cpants.cpanauthors.org/dist/Catalyst-Controller-HTML-FormFu](http://cpants.cpanauthors.org/dist/Catalyst-Controller-HTML-FormFu)

- CPAN Testers

    The CPAN Testers is a network of smokers who run automated tests on uploaded CPAN distributions.

    [http://www.cpantesters.org/distro/C/Catalyst-Controller-HTML-FormFu](http://www.cpantesters.org/distro/C/Catalyst-Controller-HTML-FormFu)

- CPAN Testers Matrix

    The CPAN Testers Matrix is a website that provides a visual overview of the test results for a distribution on various Perls/platforms.

    [http://matrix.cpantesters.org/?dist=Catalyst-Controller-HTML-FormFu](http://matrix.cpantesters.org/?dist=Catalyst-Controller-HTML-FormFu)

- CPAN Testers Dependencies

    The CPAN Testers Dependencies is a website that shows a chart of the test results of all dependencies for a distribution.

    [http://deps.cpantesters.org/?module=Catalyst::Controller::HTML::FormFu](http://deps.cpantesters.org/?module=Catalyst::Controller::HTML::FormFu)

## Bugs / Feature Requests

Please report any bugs or feature requests by email to `bug-catalyst-controller-html-formfu at rt.cpan.org`, or through
the web interface at [https://rt.cpan.org/Public/Bug/Report.html?Queue=Catalyst-Controller-HTML-FormFu](https://rt.cpan.org/Public/Bug/Report.html?Queue=Catalyst-Controller-HTML-FormFu). You will be automatically notified of any
progress on the request by the system.

## Source Code

The code is open to the world, and available for you to hack on. Please feel free to browse it and play
with it, or whatever. If you want to contribute patches, please send me a diff or prod me to pull
from your repository :)

[https://github.com/FormFu/Catalyst-Controller-HTML-FormFu](https://github.com/FormFu/Catalyst-Controller-HTML-FormFu)

    git clone https://github.com/FormFu/Catalyst-Controller-HTML-FormFu.git

# CONTRIBUTORS

- Aran Deltac <aran@ziprecruiter.com>
- bricas <brian.cassidy@gmail.com>
- dandv <ddascalescu@gmail.com>
- fireartist <fireartist@gmail.com>
- lestrrat &lt;lestrrat+github@gmail.com>
- marcusramberg <marcus.ramberg@gmail.com>
- mariominati <mario.minati@googlemail.com>
- Moritz Onken <1nd@gmx.de>
- Moritz Onken <onken@netcubed.de>
- Nigel Metheringham <nm9762github@muesli.org.uk>
- omega <andreas.marienborg@gmail.com>
- Petr Písař <ppisar@redhat.com>
