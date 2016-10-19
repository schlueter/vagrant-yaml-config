#
# This Vagrantfile is genericized and requires YAML test case files
# that look like:
#
# ---
# name: vagrant
# provisioning:
#   ansible:
#     options:
#       playbook: ../playbooks/vagrancy.yml
#       extra_vars:
#         environ_name: local
#         refinery29_scripts_git_branch: in-progress/Neptune-provisioning
#       simple_groups:
#       - refinery29
#       - vagrant
#     methods: {}
# providers:
#   virtualbox:
#     options:
#       memory: 2048
#     methods: {}
#
# This maps directly to typical Vagrantfile format, just
# more concisely and is intended for testing provisioning runs.
#

require 'yaml'


TEST_CASE_CONFIG_ENV_VAR = 'TEST_CASE_CONFIG'
MACHINE_DEFAULTS_FILE = '.test_machine_defaults.yml'


class ConfigException < StandardError; end
class MachineDefaultsFileException < StandardError; end


class ConfigFileException < StandardError
  CONFIG_FILE_ERR_MSG = <<-MSG
Please set the environment variable #{TEST_CASE_CONFIG_ENV_VAR}
to the path of a valid yaml test case file.
MSG

  def initialize(msg=nil)
    super([msg, CONFIG_FILE_ERR_MSG].join("\n\n"))
  end
end


if File.exists? MACHINE_DEFAULTS_FILE
  begin
    machine_defaults = YAML.load_file(MACHINE_DEFAULTS_FILE)
  rescue Psych::SyntaxError => syntax_error
    raise MachineDefaultsFileException, syntax_error.message
  rescue SystemCallError => system_error
    raise MachineDefaultsFileException, system_error.message
  ensure
    unless machine_defaults.class == Hash
      raise MachineDefaultsFileException, "Invalid machine defaults file. See Vagrantfile for example."
    end
  end
end

test_case_config = ENV[TEST_CASE_CONFIG_ENV_VAR]

if test_case_config == nil
  raise ConfigFileException, "No test_case file specified."
elsif not File.exists? test_case_config
  raise ConfigFileException, "Specified test case file does not exist."
else
  begin
    test_case = YAML.load_file(test_case_config)
  rescue Psych::SyntaxError => syntax_error
    raise ConfigFileException, syntax_error.message
  rescue SystemCallError => system_error
    raise ConfigFileException, system_error.message
  ensure
    unless test_case.class == Hash
      raise ConfigFileException, "Invalid test case file. See Vagrantfile for example."
    end
  end
end

unless test_case.member? 'machines'
  test_case = {'machines' => [test_case]}
end

def set_defaults_in_hash(target, defaults)
  if target
    target = target.clone
  else
    return defaults
  end
  unless defaults == nil
    defaults.each do |key, value|
      if value.class == Hash
        if target[key].class == Hash
          target[key] = set_defaults_in_hash target[key], value
      	else
      	  puts "The defaults and machine configs disagree on the clas of an item, are you sure they're correct?"
        end
      elsif target[key] == nil
        target[key] = value
      end
    end
  end
  return target
end

test_case['machines'].each_with_index do |machine, idx|
  machine.each do |key, value|
    test_case['machines'][idx] = set_defaults_in_hash(machine, machine_defaults)
  end
  if not machine.key? 'private_ip'
    raise ConfigException, 'A unique private_ip must be specified for each machine.'
  end
end

Vagrant.configure(2) do |config|
  test_case['machines'].each_with_index do |machine, idx|
    name = machine.fetch('name', "test-machine#{idx}")
    config.vm.box = machine['box']
    config.vm.host_name = machine.fetch 'host_name', name

    config.vm.network 'private_network', ip: machine['private_ip']

    if machine.key? 'provisioning'
      machine['provisioning'].each do |provisioner_name, provisioning_config|
        config.vm.provision provisioner_name do |provisioner|
          provisioning_config.fetch('options', {}).each do |key, value|
            if provisioner_name == 'ansible' && key == 'simple_groups'
              # Special case for groups so that the config is simpler
              groups =  value.each_with_object({}) do |group, hash|
                hash[group] = [ name ]
              end
              provisioner.method('groups=').call(groups)
              next
            end
            provisioner.method(key + '=').call(value)
          end
          provisioning_config.fetch('methods', {}).each do |key, value|
            provisioner.method(key).call(value)
          end
        end
      end
    end

    if machine.key? 'providers'
      machine['providers'].each do |provider_name, provider_config|
        config.vm.provider provider_name do |provider|
          provider_config.fetch('options', {}).each do |key, value|
            provider.method(key + '=').call(value)
          end
          provider_config.fetch('methods', {}).each do |key, value|
            provider.method(key).call(value)
          end
        end
      end
    end

  end
end
