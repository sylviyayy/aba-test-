# Cluster Makefile  # DO NOT remove this line!

ifndef DEBUG_ABA
.SILENT:
endif

arch   := $(strip $(shell uname -m))

TEMPLATES = ../templates
SCRIPTS   = ../scripts
debug    ?= 		# Set debug=1 for some commands will: set -x

-include ../aba.conf
platform := $(strip $(shell echo $(platform)))

# The next section is (unfortunately) needed because making a bare-metal cluster has a different flow, with interruptions!
# 1. Before iso creation, the Agent-based installer config files need to be changed: mac addresses added, disk hints ... etc.
# 2. And, after iso creation, the iso hosts need to be booted before 'aba mon' target is run.
ifneq ($(platform),vmw)   # if platform is not 'vmw', i.e. bare-metal is assumed 
	ifeq ($(wildcard .bm-message),)
		# if .bm-message DOES NOT exists
   		# Only create the agent based config files and show next steps
		TARGET_DEPENDENCIES = check ../aba.conf install-config.yaml agent-config.yaml .bm-message
	else
		#.bm-message exists
		ifeq ($(wildcard .bm-nextstep),)
			# if .bm-nextstep DOES NOT exists
   			# Only create the iso file and show next steps
			#TARGET_DEPENDENCIES = check ../aba.conf install-config.yaml agent-config.yaml             iso .bm-nextstep
			TARGET_DEPENDENCIES = check ../aba.conf                                                   iso .bm-nextstep
		else
			# .bm-nextstep exists
   			# Do all steps, including mon
			#TARGET_DEPENDENCIES = check ../aba.conf install-config.yaml agent-config.yaml             iso              mon
			TARGET_DEPENDENCIES = check ../aba.conf                                                   iso              mon
		endif
	endif
else  # If platform is 'vmw', all steps can be automated 
	# Create the agent based config files and the matching VMs
	TARGET_DEPENDENCIES = check ../aba.conf iso .autopoweroff .autoupload .autorefresh .install-complete
endif

# DEBUG # $(info TARGET_DEPENDENCIES = $(TARGET_DEPENDENCIES))

# The default target will build a cluster from scratch (for 'vmw' platform but not for bare-metal!) 
all: $(TARGET_DEPENDENCIES)

# Check cluster has been installed already and abort if it has
.PHONY: check
check:
	ln -sf ../scripts
	$(SCRIPTS)/check-cluster-installed.sh

##@ Help-related tasks
.PHONY: help
help: ## Help
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  aba \033[36mcommand\033[0m\n"} /^(\s|[\.a-zA-Z_0-9-])+:.*?##/ { printf "  \033[36m%-35s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

# aba.conf should never be missing!
../aba.conf:
	@echo "aba/aba.conf file missing! Please read the README.md on how to get started!"
	@exit 1

init: .init
.init:
	@ln -fs ../templates
	@ln -fs $(SCRIPTS)
	@ln -fs ../mirror
	@ln -fs ../mirror/regcreds 
	@ln -fs ../aba.conf
	@make -sC ../cli
	@touch .init
	@echo Initialized `pwd` successfully

mirror.conf: .init ## Set up the mirror.conf file.
	@ln -fs mirror/mirror.conf
	@touch mirror.conf
	@echo Using `pwd`/mirror.conf file 

vmware.conf: .init ## Use VMW vCenter or ESXi to create cluster.
	@make -sC ../cli govc
	@make -sC .. vmw
	@#test -f ../vmware.conf &&                          ln -fs ../vmware.conf || true
	@[ -f ../vmware.conf ] && [ ! -f vmware.conf ] && ln -s ../vmware.conf && echo vmware.conf initialized || true

cluster.conf: .init mirror.conf  ## Set up the OpenShift cluster.conf file.
	$(SCRIPTS)/create-cluster-conf.sh name=$(name) type=$(type) int_connection=$(int_connection)

agentconf: install-config.yaml agent-config.yaml  ## Convenient target to only generate the agent config files, so they can be edited. 

# Note that install-config.yaml can be configured with vSphere params (need 'vmware.conf')
install-config.yaml: .init vmware.conf cluster.conf mirror.conf  ## Create the agent-based config file based on the cluster.conf file.
	make -sC ../cli 
	$(SCRIPTS)/verify-config.sh
	$(SCRIPTS)/create-install-config.sh $(debug)

#agent-config.yaml: .init vmware.conf cluster.conf mirror.conf  ## Create the agent-based config file based on the cluster.conf file.
agent-config.yaml: .init             cluster.conf mirror.conf  ## Create the agent-based config file based on the cluster.conf file.
	$(SCRIPTS)/create-agent-config.sh $(debug)

.bm-message:
	@echo
	@echo "Next steps:"
	@echo "1 cd <cluster name>    # if not already"
	@echo "2 Check & edit the agent configuration files: "
	@echo "   - $(PWD)/install-config.yaml:"
	@echo "   - $(PWD)/agent-config.yaml:"
	@echo "       You must modify the mac addresses to match your host's network interfaces."
	@echo "       Optionally, add drive & NIC hints and any other network configurations, if needed."
	@echo "3 Run 'aba iso' to create the ISO file."
	@echo "4 Power on all your hosts and boot them with the ISO file."
	@echo "5 Verify that the hosts start up and can connect to the registry successfully."
	@echo "   - aba ssh    # connect to node0 for any needed troubleshooting."
	@echo "6 Then, run 'aba mon' to monitor the installation progress."
	@echo 
	@touch .bm-message

# The following "auto" targets are meant just for the full end-to-end automation, to build a cluster
iso: iso-agent-based/agent.$(arch).iso ## Build the agent-based iso file.
iso-agent-based/agent.$(arch).iso: install-config.yaml agent-config.yaml   # Note that install-config.yaml can itself be dependent on vmware.conf params
	$(SCRIPTS)/generate-image.sh $(debug)

.bm-nextstep:
	@echo "Next steps:"
	@echo "1 Boot your hosts with the ISO file."
	@echo "2 if needed, run 'aba ssh' to log into node0 to troubleshoot."
	@echo "3 then run 'aba mon' to monitor the installation progress."
	@touch .bm-nextstep
	@exit 0

.autopoweroff: iso-agent-based/agent.$(arch).iso
	$(SCRIPTS)/vmw-kill.sh $(debug) || true 
	@touch .autopoweroff

upload: .autoupload  ## Upload the ISO to the VMW platform to [Datastore] images/agent-<cluster-name>.iso
#.autoupload: vmware.conf .autorefresh
.autoupload: vmware.conf .autopoweroff
	$(SCRIPTS)/vmw-upload.sh $(debug) 
	@touch .autoupload


# The following targets are meant for interactive use only

# This target is a duplicate of .autorefresh below, except for the touch command.  'aba refresh' is for interactive use and will always run.
.PHONY: refresh
refresh: vmw agentconf  ## Delete & create the VMs. This will automatically start the cluster installation again which can be useful when your cluster is broken and you want to redeploy it.
	$(SCRIPTS)/vmw-refresh.sh workers=$(workers) masters=$(masters) 

# This target is the same as 'refresh' but should only run once depending on .init .autoupload & vmware.conf
.autorefresh: .init .autoupload vmware.conf
	$(SCRIPTS)/vmw-refresh.sh
	@touch .autorefresh

.PHONY: create
create: vmw agentconf      ## Create the VMs based on the 'agent-based' config files. Use 'aba start' to start them. 
	$(SCRIPTS)/vmw-exists.sh || $(SCRIPTS)/vmw-create.sh $(debug) && exit 0

###ls: vmware.conf  ## List VMs and their power state.
.PHONY: ls
ls: vmw                           agent-config.yaml ## List all VMs and their power state.
	$(SCRIPTS)/vmw-ls.sh || echo "No vm(s)."

.PHONY: shutdown 
shutdown: ## Trigger the cluster to gracefully shut itself down by draining all the nodes of pods and running shutdown command.
	$(SCRIPTS)/cluster-graceful-shutdown.sh 

.PHONY: startup
#startup: vmware.conf ## Start all VMs and allow all pods to start (uncorden).  Start up your bare-metal servers and run this target.
startup:             ## Start all VMs and allow all pods to start (uncorden).  Start up your bare-metal servers and run this target.
	$(SCRIPTS)/cluster-startup.sh 

.PHONY: rescue
rescue: ## If a cluster does not start due to login/CSR issues, rescue the cluster using ssh, if available. 
	$(SCRIPTS)/cluster-rescue.sh 

# $(wait) $(workers) are set in aba.sh 
.PHONY: stop
stop: .init vmware.conf                     agent-config.yaml ## Shut down all VMs gracefully. This will stop the cluster. Be warned that clusters don't always start up correctly again! Try using 'aba shutdown' for graceful cluster shutdown.
	$(SCRIPTS)/vmw-stop.sh wait=$(wait) workers=$(workers) masters=$(masters) debug=$(debug)

.PHONY: start
start: .init vmware.conf                     agent-config.yaml ## Power on all VMs or limit by workers=1 or masters=1.  Use 'aba startup' to ensure all nodes are uncordoned. 
	$(SCRIPTS)/vmw-start.sh debug=$(debug) workers=$(workers) masters=$(masters) || exit 0

.PHONY: poweroff
poweroff: .init vmware.conf                     agent-config.yaml ## Power off all VMs immediately.  This will stop the cluster abruptly. 
	$(SCRIPTS)/vmw-kill.sh $(debug) || exit 0

.PHONY: kill
kill:  .init vmware.conf                     agent-config.yaml ## Kill or power off the VMs immediately.  This will stop the cluster abruptly. 
	$(SCRIPTS)/vmw-kill.sh $(debug) || exit 0

.PHONY: delete
delete: .init vmware.conf                     agent-config.yaml ## Delete all VMs. This will delete the cluster!
	$(SCRIPTS)/vmw-delete.sh $(debug) || exit 0

.install-complete: .autorefresh
	$(SCRIPTS)/monitor-install.sh $(debug)
	@touch .install-complete

.PHONY: mon
# install and mon and the same
install: mon  ## Install the cluster using the cluster.conf file
mon:  ## Monitor the agent-based installation of OpenShift.
	$(SCRIPTS)/monitor-install.sh $(debug)
	@touch .install-complete

.PHONY: bootstrap
# monitor for bootstrap complete 
bootstrap:  ## Monitor the agent-based bootstrap installation of OpenShift.
	$(SCRIPTS)/monitor-bootstrap.sh $(debug)
	#@touch .install-complete

.PHONY: info
info:  ## Show cluster access credentials 
	$(SCRIPTS)/cluster-info.sh $(debug)

.PHONY: login
login: 
	$(SCRIPTS)/show-cluster-login.sh

.PHONY: getco
getco: ## Run 'oc get co' command on the cluster.
	oc --kubeconfig iso-agent-based/auth/kubeconfig get co

.PHONY: cmd
cmd: ## Run any oc command on the cluster. Must provide command, e.g. cmd="oc get pods -A", default is 'oc get co'.
	$(SCRIPTS)/oc-command.sh $(cmd)

.PHONY: ssh
ssh: ## Open ssh to the rendezvous host (node 0) for troubleshooting
	$(SCRIPTS)/ssh-rendezvous.sh $(cmd)

.PHONY: shell   
shell:  ## Gain access to the installed cluster.  Run '. <(aba -s shell)'
	@echo "export KUBECONFIG=$(CURDIR)/iso-agent-based/auth/kubeconfig"

.PHONY:
day2:  ## Set up OperatorHub, CatalogSources, imageContentSourcePolicy resources. Run this target after new Operators are loaded into the mirror registry. Requires access to 'mirror/.../oc-mirror-workspace' directory!
	$(SCRIPTS)/day2.sh $(debug)

.PHONY:
day2-ntp:  ## Configure OpenShift with your NTP server
	$(SCRIPTS)/day2-config-ntp.sh $(debug)

.PHONY:
day2-osus:  ## Install OpenShift Update Service and integrate with the Admin Console
	$(SCRIPTS)/day2-config-osus.sh $(debug)

.PHONY: remove-cdrom
remove-cdrom: vmware.conf  ## Remove the cdrom devise from all VMs
	$(SCRIPTS)/vmw-remove-cdrom-device.sh

.PHONY: vmw
vmw: vmware.conf ## Configure vCenter/ESXi API credentials in vmware.conf file.
	#@[ ! -s ../vmware.conf ] && rm -f ../vmware.conf || true
	#@make -sC .. vmw

.PHONY: noask
noask:  ## Set 'ask' in aba.conf to 'false'.  Aba will assume all defaults and not prompt you for anything which allows for fully automated installations.
	@make -sC .. noask

.PHONY: ask
ask: ## Set 'ask' in aba.conf to 'true'  Aba will prompt you for certain actions. 
	@make -sC .. ask

# Note that only delete 'vmware.conf/regcreds/aba.conf' if they are symbolic links (i.e. not files/changed by user) 
.PHONY: clean
clean: ## Clean up files generated by 'aba', except your own configuration files. 'aba clean && aba' should re-build your cluster from scratch, using the same cluster.conf file. It is recommended to backup your important files first!
	rm -f install-config.yaml agent-config.yaml 
	rm -rf iso-agent-based*
	rm -f .??*
	rm -f mirror
	rm -f mirror.conf
	@[ -L regcreds ] && rm regcreds || true
	@[ -L vmware.conf ] && rm vmware.conf || true
	@[ -L aba.conf ] && rm aba.conf  || true
	rm -f scripts templates
	rm -f openshift-install-*-*

.PHONY: reset
reset: # Clean up all files.  Only use if you know what you're doing! Only used to re-distribute the repository.
	@[ -x $(SCRIPTS)/reset-gate.sh ] && $(SCRIPTS)/reset-gate.sh $(force) || true
	@make clean
	rm -f cluster.conf mirror.conf vmware.conf
	rm -f cluster.conf aba.conf                  vmware.conf                *.backup

